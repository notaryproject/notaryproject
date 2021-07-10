# Design proposals

This document is meant to capture a few different ideas around how to relate
signatures to the objects we wish to sign.  This is meant as a discussion
point, and not as a concrete proposal to use a particular approach (or even a
recommendation).  This document is *not* exhaustive, and is meant as a companion
to both @SteveLasker's [scenarios pull request][stevelasker-scenarios] and
@justincormack's [signing overview][jc-signing-overview].

## Terms

Options here make use of the following terms:

* [OCI Image Manifest][oci-manifest]: A description of a container image,
  including the constituent layers and configuration as content-addressable
  references ([descriptors][oci-descriptor]).
* [OCI Descriptor][oci-descriptor]: A structure describing content, including
  the media type, a content-addressable digest, the size, and other properties.
  Descriptors are used to describe layers and configuration in a manifest.
* [OCI Image Index][oci-image-index]: A higher-level manifest which points to
  specific [image manifests][oci-manifest], typically used to describe
  platform-specific (e.g., architecture- and operating-system specific)
  images that can be identified collectively and referred to together.  The
  specific image manifiests are identified by modified
  [descriptors][oci-descriptor] with additional properties and restrictions.
* [OCI Annotations][oci-annotations]: A key-value map that can be associated
  with [desciptors][oci-descriptor] and [image manifests][oci-manifest].
* Subject: The data that is signed.
* [Fingerprint][fingerprint]: a short identifier of a given public key.

## Concerns

In the discussions about signing, including in the Seattle kick-off meeting, in
the ongoing phone calls, and in @SteveLasker's [scenarios pull
request][stevelasker-scenarios] , several different areas of focus have been
identified as possibly desirable.  These include:

* Storing the signature alongside the content
* Identifying the signer
* Ensuring the signature is not tied to a particular name (e.g., support
  renames without breaking the signature) or to a particular remote location
  (i.e, a registry)
* Identifying some canonical metadata about an image, including name and/or
  version in a verifiable manner
* Identifying the content of the image, in some machine-parsable and verifiable
  way
* Distributing keys
* Support multiple signatures over the same content


## Option 1a: OCI Image Index with detached signature manifests

This was originally @jonjohnsonjr's "strawman" proposed at the Notary v2
kickoff meeting in Seattle.

This option proposes adding a new top-level descriptor to the existing [OCI
Image Index][oci-image-index] that points to raw signature data and is
[annotated][oci-annotations] with the digest of the image manifest to which the
signature applies.

```json
{
  "manifests": [
    {
      "digest": "sha256:134c7fe821b9d359490cd009ce7ca322453f4f2d018623f849e580a89a685e5d",
      "mediaType": "something-something-x509-signature-mediaType",
      "size": 1337,
      "annotations": {
        "org.opencontainers.image.signature.subject": "sha256:349e3988c0241304b39218794b8263325f7dc517317e00be37d43c3bdda9449b",
        "org.opencontainers.image.signature.fingerprint": "43:51:43:a1:b5:fc:8b:b7:0a:3a:a9:b1:0f:66:73:a8"
      }
    },
    {
      "digest": "sha256:349e3988c0241304b39218794b8263325f7dc517317e00be37d43c3bdda9449b",
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 1152
    }
  ],
  "schemaVersion": 2
}
```

### Implications

* The signed content is the platform-specific manifest, so each platform may
  have a separate signature.
* There can be multiple signatures attached to each manifest in the index.
* Adding a new signature means creating a new index.  As the index is also
  content-addressable, this creates a new digest.

### Not covered

* Key distribution
* Expiration
* Identity of the image ([Justin Cormack's][jc-signing-overview] "git model")
* Image content ([Justin Cormack's][jc-signing-overview] "git model")

## Option 1b: OCI Image Index with signature in manifest descriptor

This was originally @vbatts's response to @jonjohnsonjr's "strawman" option
above.  It is largely similar, with the only difference being that the signature
descriptor is embedded inside a new "signatures" block in the specific manifest
descriptor to which the signature applies.

```json
{
  "manifests": [
    {
      "digest": "sha256:349e3988c0241304b39218794b8263325f7dc517317e00be37d43c3bdda9449b",
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 1152,
      "signatures": [
        {
          "digest": "sha256:134c7fe821b9d359490cd009ce7ca322453f4f2d018623f849e580a89a685e5d",
          "mediaType": "something-something-x509-signature-mediaType",
          "size": 1337,
          "annotations": {
            "org.opencontainers.image.signature.fingerprint": "43:51:43:a1:b5:fc:8b:b7:0a:3a:a9:b1:0f:66:73:a8"
          }
        }
      ]
    }
  ],
  "schemaVersion": 2
}
```

### Implications

* The signed content is the platform-specific manifest, so each platform may
  have a separate signature.
* There can be multiple signatures attached to each manifest in the index.
* Adding a new signature means creating a new index.  As the index is also
  content-addressable, this creates a new digest.

### Not covered

* Key distribution
* Expiration
* Identity of the image ([Justin Cormack's][jc-signing-overview] "git model")
* Image content ([Justin Cormack's][jc-signing-overview] "git model")

## Option 2a: OCI descriptor with canonical name and version

This approach is a modification of either 1a or 1b with the addition of
annotations that cover the identity and/or version of the image.  The purpose
of these annotations are to prevent substitution of an old version with a valid
signature.

```json
{
  "digest": "sha256:349e3988c0241304b39218794b8263325f7dc517317e00be37d43c3bdda9449b",
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "size": 1152,
  "annotations": {
    "org.opencontainers.image.canonical-name": "foo",
    "org.opencontainers.image.canonical-name.version": "1.0.0-dev+bar"
  }
}

```

Or, with properties:

```json
{
  "digest": "sha256:349e3988c0241304b39218794b8263325f7dc517317e00be37d43c3bdda9449b",
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "size": 1152,
  "canonicalName": "foo",
  "canonicalVersion": "1.0.0-dev+bar"
}
```

The signer can place annotations into the descriptor of the manifest, and as
long as the annotations are included in the input to the signature, they can be
cryptographically verified.  A client wishing to validate an image that it
pulls can both validate the signature and inspect the annotations to ensure
they are as expected.

### Implications

* The signed content is the platform-specific manifest, so each platform may
  have a separate signature.
* There can be multiple signatures attached to each manifest in the index.
* Adding a new signature means creating a new index.  As the index is also
  content-addressable, this creates a new digest.
* Manifests in the same index may have different canonical names/versions.
* The name and version need to be signed.  JSON can be formatted in different
  ways, structures and maps are not necessarily ordered or have an ordering that
  is preserved by parsing software.  Some canonical formatting and ordering
  would need to be adopted to sign these, as they are not an arbitrary blob of
  data.  ([TUF][tuf] uses the OLPC projet's [Canonical JSON][canonical-json]
  for this.)

### Not covered

* Key distribution
* Expiration/revocation
* Full image content ([Justin Cormack's][jc-signing-overview] "content model")

## Option 2b: OCI descriptor with image content

This approach is similar to 2a, but with the idea that we'd like to encode
additional information about the content of the image into the signed data as
well.

```json
{
  "digest": "sha256:349e3988c0241304b39218794b8263325f7dc517317e00be37d43c3bdda9449b",
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "size": 1152,
  "annotations": {
    "org.opencontainers.image.component.foo": "1.2.3-456",
    "org.opencontainers.image.component.bar": "3.2.1~distro"
  }
}
```

Or, a different layout:

```json
{
  "digest": "sha256:349e3988c0241304b39218794b8263325f7dc517317e00be37d43c3bdda9449b",
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "size": 1152,
  "components": [
    {
      "name": "foo",
      "version": "1.2.3-456"
    },
    {
      "name":"bar",
      "version": "3.2.1~distro"
    }
  ]
}
```

### Implications
* The signed content is the platform-specific manifest, so each platform may
  have a separate signature.
* There can be multiple signatures attached to each manifest in the index.
* Adding a new signature means creating a new index.  As the index is also
  content-addressable, this creates a new digest.
* Manifests in the same index may have different components.
* The components need to be signed.  JSON can be formatted in different ways,
  structures and maps are not necessarily ordered or have an ordering that is
  preserved by parsing software.  Some canonical formatting and ordering would
  need to be adopted to sign these, as they are not an arbitrary blob of data.
  ([TUF][tuf] uses the OLPC projet's [Canonical JSON][canonical-json] for this.)
* A long list of components balloons the size of the descriptor, and a client
  must download the whole thing in order to validate the signature.
* Now we're responsible for designing a way to describe image components, rather
  than just a signing mechanism and storage layout.

### Not covered

* Key distribution
* Expiration/revocation

## Option 3: OCI Index with Software Bill-of-Materials (SBOM) artifact

Use a Software Bill-of-Materials to describe the content of the image.  Sign
the SBOM and sign the image content as well.  The signature would need to be
over a normalized composite of the two; so maybe that means we order the
descriptors of both the SBOM and the manifest so that the client can validate
the signature.

```json
{
  "manifests": [
    {
      "digest": "sha256:134c7fe821b9d359490cd009ce7ca322453f4f2d018623f849e580a89a685e5d",
      "mediaType": "something-something-x509-signature-mediaType",
      "size": 1337,
      "annotations": {
        "org.opencontainers.image.signature.subject": "sha256:349e3988c0241304b39218794b8263325f7dc517317e00be37d43c3bdda9449b",
        "org.opencontainers.image.signature.bom": "sha256:8bdbff2759805dc62c3f0e447f81d288b98c25e4a551cda3daefc7976b6c1ca3",
        "org.opencontainers.image.signature.fingerprint": "43:51:43:a1:b5:fc:8b:b7:0a:3a:a9:b1:0f:66:73:a8"
      }
    },
    {
      "digest": "sha256:349e3988c0241304b39218794b8263325f7dc517317e00be37d43c3bdda9449b",
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 1152
    },
    {
      "digest" : "sha256:8bdbff2759805dc62c3f0e447f81d288b98c25e4a551cda3daefc7976b6c1ca3",
      "mediaType": "something-something-SBOM-mediaType",
      "size": 756
    }
  ],
  "schemaVersion": 2
}
```

Or, an alternate:

```json
{
  "manifests": [
    {
      "digest": "sha256:349e3988c0241304b39218794b8263325f7dc517317e00be37d43c3bdda9449b",
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 1152,
      "signatures": [
        {
          "digest": "sha256:134c7fe821b9d359490cd009ce7ca322453f4f2d018623f849e580a89a685e5d",
          "mediaType": "something-something-x509-signature-mediaType",
          "size": 1337,
          "fingerprint": "43:51:43:a1:b5:fc:8b:b7:0a:3a:a9:b1:0f:66:73:a8"
        }
      ],
      "bom": {
        "digest" : "sha256:8bdbff2759805dc62c3f0e447f81d288b98c25e4a551cda3daefc7976b6c1ca3",
        "mediaType": "something-something-SBOM-mediaType",
        "size": 756
      }
    }
  ],
  "schemaVersion": 2
}
```

### Implications
* The signed content is the platform-specific manifest, so each platform may
  have a separate signature.
* There can be multiple signatures attached to each manifest in the index.
* Adding a new signature means creating a new index.  As the index is also
  content-addressable, this creates a new digest.
* Manifests in the same index may have different components.
* The SBOM needs to be signed as well.  To do so, a canonical way to include the
  SBOM in the signed content will need to be declared.
* A large SBOM means a client needs to download the entire SBOM to properly
  validate the signature.  (A client may instead decide to partially-validate
  by instead trusting the digest in the descriptor rather than downloading the
  content.)
* Notary v2 now depends on the SBOM spec, which is not complete and not
  necessarily fully aligned with the goals of Notary v2.

### Not covered

* Key distribution
* Expiration/revocation

## Option 4: Arbitrary Claims

We can enable signatures to cover arbitrary descriptors in the OCI manifest and
allow those descriptors to be "claims" of the signature.  This would allow us to
sign any kind of content (including things like SBOM), combinations of content,
and have different signatures that include different claims.

```json
{
  "manifests": [
    {
      "digest": "sha256:349e3988c0241304b39218794b8263325f7dc517317e00be37d43c3bdda9449b",
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 1152
    },
    {
      "digest" : "sha256:8bdbff2759805dc62c3f0e447f81d288b98c25e4a551cda3daefc7976b6c1ca3",
      "subjects": [
        "sha256:349e3988c0241304b39218794b8263325f7dc517317e00be37d43c3bdda9449b"
      ],
      "mediaType": "something-something-SBOM-mediaType",
      "size": 756
    },
    {
      "digest": "sha256:134c7fe821b9d359490cd009ce7ca322453f4f2d018623f849e580a89a685e5d",
      "mediaType": "something-something-x509-signature-mediaType",
      "size": 1337,
      "subjects":[
        "sha256:349e3988c0241304b39218794b8263325f7dc517317e00be37d43c3bdda9449b",
        "sha256:8bdbff2759805dc62c3f0e447f81d288b98c25e4a551cda3daefc7976b6c1ca3"
      ],
      "fingerprint": "43:51:43:a1:b5:fc:8b:b7:0a:3a:a9:b1:0f:66:73:a8"
    },
    {
      "digest": "sha256:4e18f16c238c024b75528cedb710a95898d37c492d9204bda8983c23a2d3d517",
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 8033
    },
    {
      "digest": "sha256:26630bbf984731850268a398cbd47e30fbecd0ac06afac0338369a0e7a95be00",
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 5050
    },
    {
      "digest" : "sha256:8bdbff2759805dc62c3f0e447f81d288b98c25e4a551cda3daefc7976b6c1ca3",
      "subjects": [
        "sha256:4e18f16c238c024b75528cedb710a95898d37c492d9204bda8983c23a2d3d517",
        "sha256:26630bbf984731850268a398cbd47e30fbecd0ac06afac0338369a0e7a95be00"
      ],
      "mediaType": "something-something-SBOM-mediaType",
      "size": 867
    },
    {
      "digest": "sha256:24c41678dd21e48094b108302541316ad580e1d4919bc6f3d743871aa55266e7",
      "mediaType": "something-something-x509-signature-mediaType",
      "size": 1337,
      "subjects":[
        "sha256:4e18f16c238c024b75528cedb710a95898d37c492d9204bda8983c23a2d3d517",
        "sha256:26630bbf984731850268a398cbd47e30fbecd0ac06afac0338369a0e7a95be00",
        "sha256:8bdbff2759805dc62c3f0e447f81d288b98c25e4a551cda3daefc7976b6c1ca3"
      ],
      "fingerprint": "ee:51:43:a1:b5:ee:8b:b7:0a:3a:a9:b1:0f:66:73:a8"
    },
    {
      "digest": "sha256:159f81ef714d0f57df8c909bb3863e9cb943bcfd0abe178bb86807ea716e0e60",
      "mediaType": "something-something-x509-signature-mediaType",
      "size": 1337,
      "subjects":[
        "sha256:349e3988c0241304b39218794b8263325f7dc517317e00be37d43c3bdda9449b",
        "sha256:4e18f16c238c024b75528cedb710a95898d37c492d9204bda8983c23a2d3d517",
        "sha256:26630bbf984731850268a398cbd47e30fbecd0ac06afac0338369a0e7a95be00"
      ],
      "fingerprint": "ff:51:43:a1:b5:42:8b:b7:0a:3a:a9:b1:0f:66:82:ff"
    },

  ],
  "schemaVersion": 2
}
```

This example shows:
* 3 manifests
* 2 SBOMs, 1 which describes multiple manifests
* 3 signatures, covering
  1. A pair of manifest + SBOM
  2. The other two manifests + SBOM
  3. All three manifests, but no SBOM

### Implications
* A signature can cover a platform-specific manifest, an SBOM, some other type
  of object, or combinations of those.
* Each platform may have a separate signature.  A signature can also cover
  multiple platforms.
* There can be multiple signatures attached to each manifest in the index.
* Adding a new signature means creating a new index.  As the index is also
  content-addressable, this creates a new digest.
* Manifests in the same index may have different components.
* The associated subjects need to be signed as well.  To do so, a canonical way
  to include the subject in the signed content will need to be declared.
* A large subject means a client needs to download the entire data described by
  the referenced descriptors properly validate the signature.  (A client may
  instead decide to partially-validate by instead trusting the digest in the
  descriptor rather than downloading the content.)
* Clients need to understand how to associate signatures and subjects together
* A policy implementation will need to specify how to decide whether a given
  signature is trusted.
* Notary does not depend on the SBOM spec.

### Not covered

* Key distribution
* Expiration/revocation

## Option 5: TBD

Let's brainstorm more options as well.

[oci-manifest]:          https://github.com/opencontainers/image-spec/blob/master/manifest.md
[oci-descriptor]:        https://github.com/opencontainers/image-spec/blob/master/descriptor.md
[oci-image-index]:       https://github.com/opencontainers/image-spec/blob/master/image-index.md
[oci-annotations]:       https://github.com/opencontainers/image-spec/blob/master/annotations.md
[jc-signing-overview]:   https://docs.google.com/document/d/1lffOYCDXBoxRumQre32yv1jUN2NfOkWugoDEJ_2DdEk/edit#
[stevelasker-scenarios]: https://github.com/notaryproject/requirements/pull/1
[tuf]:                   https://github.com/theupdateframework/specification/blob/master/tuf-spec.md
[canonical-json]:        http://wiki.laptop.org/go/Canonical_JSON
[fingerprint]:           https://en.wikipedia.org/wiki/Public_key_fingerprint
