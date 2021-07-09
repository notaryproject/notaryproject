# Overview
Key management for container signing can be broadly categorized into three general use cases:
- Key Setup/Signing (Key managment and integrations with client tooling for generating signatures.)
- Deployment configurations (Configuring trust policies and which keys to trust for artifacts.)
- Signature Validity (Expiry, Revocation and/or Trusted Update List. Mechanism to update information on the validity of signatures.)

## Personas:
- Publisher: User who builds and signs containers.
    - Publisher Admin: In some scenarios a publisher will include a group of users (i.e. teams or enterprises). Admin users (i.e. security administrator) will be responsible for configuring roots, provide access to use or generate delegate keys, and make decisions for key revocation.
    - Publisher Builder: In some scenarios a publisher will include a group of users (i.e. teams or enterprises). Builders (i.e. developers, build hosts) will be responsible for creating artifacts. 
    - Publisher Signer: In some scenarios a publisher will include a group of users (i.e. teams or enterprises). Signers (i.e. developers, signing hosts) will be responsible for signing artifacts.
- Deployer: User who deploys containers.
    - Deployer Admin: In some scenarios a deployer will include a group of users (i.e. teams or enterprises). Admin users (i.e. security administrators) will be responsible for configuring deployment policies, including potentially a set of trusted roots or required signatures. 
    - Deployer Operator: In some scenarios a publisher will include a group of users (i.e. teams or enterprises). Operators (i.e. developers, deployment hosts) will be responsible for pulling container images from registries, verifying their signatures, and then running them.
- Repository Owner: In some scenarios a container may be stored in a repository that is managed by the repository owner. The repository may be public or private and could also be air-gapped.

## Definitions:
- Root key: A self signed key used for the lowest designation of trust. Root keys can be created by developers, organizations, public/private CAs, and registry operators. The root key should be retrieved from a trusted source that can establish the authenticity of the creator's identity.
- Signing key: A signing key is used to generate artifact signatures. A signing key should be signed with a root key or one of its intermediaries. The certificate chain with a signing key can be used to verify which root it belongs to. While a root key can be used as a signing key, this is not recommended as it creates a large blast radius and increases the risk of compromising a root key. 
- Trust Policy: The trust policy defines whether to enable signature validation and which checks need to be run. An example trust policy:
```
    {
        "trustPolicy": {
            "signatureCheck": true,
            "optionalChecks": {
            	"signatureExpiry": true,
            	"signatureRescinded": false
            },
            "trustStore": [
            	"truststore.json",
            	{
            		"trustedRoots": [{
            			"root": "test.crt"
            		}]
            	}
            ],
            "trustedArtifacts": []
        }
    }
```
- Trust Store: The trust store defines the relationship between signing keys and artifacts that are used at validation time to determine whether to trust an artifact with a crytptographically valid signature. The trust store will relate a scope (any source, specific registry, specific repository, or specific target) with a certificate (for root key, intermediate, or signing key) or key repository (for automated key distribtuion). It is not recommended to use a signing key as this will cause signature validation to fail if the signing key is rotated. An example trustStore where the first root is trusted for artifacts from any registry and the second root is only trusted for artifacts from "registry.wabbit-networks.io" :
```
    {
    	"trustedRoots": [{
 	    		"root": "-----BEGIN CERTIFICATE-----examplecertificate-----END CERTIFICATE-----"
 	    	},
 	    	{
 	    		"root": "-----BEGIN CERTIFICATE-----examplecertificate-----END CERTIFICATE-----",
 	    		"scope": "registry.wabbit-networks.io"
 	    	}
 	    ]
    }
```
- Trust Store Key Information: The key information configured in the trust store will be cryptographically verifiable and contain:
    - Public Key: Will be used to verify signed artifacts.
    - Signed Identity: Identity of creator of the signing key.
    - (Optional) Issuer: If the key is an intermediate this will describe the owner of the root.
    - (Optional) Mechanism to check whether signatures generated with the signing key have been rescinded.

## Requirements
- Signing an artifact MUST NOT require the publisher to perform additional actions with a registry/repository or registry operator. Uploading a signature MAY require additional actions.
- Retrieving a signature MUST NOT require the deployer to perform additional actions with a registry/repository or registry operator beyond those required to pull an unsigned artifact.
- Artifact integrity, source, and signature expiry MUST be verifiable from the signature AND NOT require additional calls to the registry/repository.
- Signature allow list/deny list MAY require additional calls that will be defined in a separate document.
- Copying an artifact from one repository to another SHOULD NOT invalidate the signature on the artifact.
- Publishers SHOULD be able to sign with keys stored on their local machines, secure tokens, Hardware Security Modules (HSMs), or cloud based Key Management Services.
- Publishers SHOULD be able to generate multiple signatures for a single artifact.
- Publisher admins MUST have a mechanism to revoke signatures to indicate they are no longer trusted. 
- Trust stores MUST be configurable by the deployer admin.
- Deployer admins MUST be able to configure trusted entities for individual repositories and targets.
- Deployers MUST be able to validate signatures on any version of an artifact including whether they have been revoked by the publisher.
- Signature validation MUST be enforceable in air-gapped environments. 

## Prototype Stages
1. Signature generation for each of the key storage scenarios. A succesful prototype should enable signing with keys on each of the following: local host, secure tokens, Hardware Security Modules (HSMs), and cloud based Key Management Services.
2. Trust store configuration and signature source validation in runtime environments. A succesful prototype should enable configuring a trust store in a runtime environment. The prototype should validate signatures from a trusted source and reject signatures from a source that is not listed. The prototype will not check for other aspects of signature validity (expiry/revocation).
3. Key rotation. A succesful prototype should meet all requirements from the key rotation document.
4. Signature/Key Expiration. A succesful prototype should meet all requirements from the signature/key expiry document.
5. Signature allowlist/denylist. A succesful prototype should meet all requirements from the signatur allowlist/denylist document.
