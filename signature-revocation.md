# Signature Revocation

Signatures ensure the authenticity and integrity of the signed content but not its quality. One of the goals of Notary v2 is to build in solutions for signature revocation that publishers can ensure the quality of the content distributed to users. This document discusses some potential mechanisms for signature revocation.

In general, signatures are invalidated by expired or revoked keys. The approaches to revoke signatures signed by active keys are similar to [key revocation](key-revocation.md). In this document, signature expiration is focused.


## Signature Expiration

Signed content can have different timeliness, which can be shorter than its signing key. Adding an expiration time to a signature allows it to be revoked automatically before the corresponding signing key being revoked. For instance, artifact publishers MAY publish artifacts with various life-cycle. Some artifacts are in preview and have short-term support. Others are generally available (GA) and have long-term support (LTS). Therefore, configurable signature expiry is desirable for individual artifacts. In this section, artifact signatures are focused where the signed content is the hash of the artifact.

An added benefit is to defend against [freeze attack](https://doi.org/10.1145/1455770.1455841). Since the signature expires in a short time although the signing key can live longer, verifiers have to obtain the latest signature to verify, which implies that verifies have access to the latest content.

Pros:
- Signatures can have shorter expiry time than the signing key
    - The life-cycle of the artifacts are fine-controlled by the publishers
    - Freeze attack is defended
- Artifact support timeline is indicated by the signature expiry
    - The deployers can choose the most suitable version in advance
- Healthier software ecosystem
    - Publishers can safely retire older artifact releases

Cons:
- Publishers have to sign the artifact more often

### Scenarios

With explicit signature expiry in addition to the key expiry, publishers are able to control the life-cycle of various published artifacts at a fine-grained level.

Artifacts can be categorized into
- Nightly built artifacts
- Preview artifacts
    - Instances are `alpine/helm:3.5.0-rc.1`, `python:3.10-rc` ...
- Current artifacts
    - Instances are `ubuntu:20.10`, `ubuntu:21.04` ...
- Artifacts with long-term support (LTS)
    - Instances are `ubuntu:20.04`, `mcr.microsoft.com/dotnet/sdk:3.1` ...

The expiry of a artifact signature implies that the artifact is no longer supported by the publisher. On reaching the signature expiry, artifact consumers have the following options:

- Find artifacts with valid signatures
    - Update to a newer version
    - Fallback to an older version
        - It is possible since `ubuntu:19.10` reached the end of life on July 17, 2020 but users can still fallback to `ubuntu:18.04` since it is supported till April 2028.
    - Ask the publisher for a new signature with extended expiry
- Stop using artifacts and optionally find alternatives
- Re-sign the artifact using consumers own keys with caution
    - The resulted signature may not be trusted by others
- Suppress the signature verification (insecure)

Here is an example how signature expiry helps both publishers and deployers stay in a healthy ecosystem.

1. A publisher wants to release an artifact `net-monitor:v1`. Thus it publishes an release candidate `net-monitor:v1-rc` on `2021-04-02T08:00:00Z` with signature expiry on `2021-05-01T00:00:00Z`.
2. A deployer picks up `net-monitor:v1-rc` on `2021-04-10T12:34:56Z`, and verifies its signature regularly. Since the current time `2021-04-10T12:34:56Z` is before the signature expiry `2021-05-01T00:00:00Z`, the signature is valid and the artifact is accepted.
3. The publisher later fixes bugs in `net-monitor:v1-rc`, including security vulnerability fixes, and releases `net-monitor:v1` on `2021-04-15T08:00:00Z` with signature expiry on `2022-05-01T00:00:00Z`.
4. After 1 month, the deployer continues to deploy `net-monitor:v1-rc` on `2021-05-10T12:34:56Z`. Since the current time `2021-05-10T12:34:56Z` is after the signature expiry `2021-05-01T00:00:00Z`, the signature is invalid. Thus the deployment is failed and an alert is sent to the deployer admin.
5. The deployer admin updates the artifact version from `v1-rc` to `v1` either manually or automatically.
6. The deployer picks up `net-monitor:v1` on `2021-05-11T12:34:56Z`, and verifies its signature regularly. Since the current time `2021-05-11T12:34:56Z` is before the signature expiry `2022-05-01T00:00:00Z`, the signature is valid and the artifact is accepted.

Without signature expiry, the deploy may stick with `net-monitor:v1-rc` till the signing key expires, which could be several years later.

1. A publisher wants to release an artifact `net-monitor:v1`. Thus it publishes an release candidate `net-monitor:v1-rc` on `2021-04-02T08:00:00Z`.
2. A deployer picks up `net-monitor:v1-rc` on `2021-04-10T12:34:56Z`, verifies its signature regularly, and accepts the artifact on its validity.
3. The publisher later fixes bugs in `net-monitor:v1-rc`, including security vulnerability fixes, and releases `net-monitor:v1` on `2021-04-15T08:00:00Z`.
4. After 1 month, the deployer continues to deploy `net-monitor:v1-rc` on `2021-05-10T12:34:56Z`. Since the signature for `net-monitor:v1-rc` is still valid, the deployer proceeds without alerts.

At this point, the deployer always uses a version without bug fix. The publisher may not be aware of that `net-monitor:v1-rc` is still in use even after several years. Since using the release candidate `v1-rc` for a long time is harmful to the software ecosystem, all players enter a no-win situation.

### Recommendation

The artifact signature SHOULD have signature expiry. If the signature expiry does not present, it is implied by the key expiry.
