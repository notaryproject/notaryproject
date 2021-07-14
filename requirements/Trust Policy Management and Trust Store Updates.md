# Trust Policy Management and Trust Store Updates 

Deployers who consume signed artifacts from a registry require that only artifacts from trusted parties are deployed and executed. The trusted parties are specified in some form of a trust policy against which the signed artifacts are validated. The trust policy can include a trust store with trusted root and leaf keys/certificates which represent the Publisher identities the Deployer trusts. This section covers where the trust policies should be stored, and how they are distributed/updated.

Root keys form the basis of hierarchical trust systems, where intermediate and leaf keys chain back to a root key, and leaf keys are used to generate signatures. Consumers of signed artifacts associate a baseline level of trust with signatures that chain to roots they trust. In the scenario that a root key is no longer trusted, all signed artifacts chaining back to the root are no longer trusted. The conditions which render the root keys to be no longer trusted, are disclosed to the customer through some other channel (CVE, internal security audit, etc.). Safe rotation of keys require a overlap period where both new and old root keys are trusted. This is always not possible, such as when a root key is compromised.

## Scenarios
1. A Deployer consumes artifacts from a third party Publisher and wants to configure the trusted publishers in a policy. The Deployer is not the same user/organization as Publisher and wants to independently define the trusted publishers.
2. A Publisher’s repository may contain a mix of artifacts, published by different teams using different keys. The Deployer wants to trust only artifacts signed by specific keys.
    - E.g. A group may include the MySQL image from docker hub, and the MySQL Helm chart from another group in the same registry/repo. These are signed by different entities, and valid to be placed in the same repository.
    
#### Scenarios for Trust Store Updates
1. A Deployer updates their dependencies and no longer consumes artifacts from particular Publisher
2. A public root associated with a signing key a Publisher uses to sign artifacts is compromised.
3. An root key used by a Publisher is close to expiry. A new key is created and rotated before the existing root expires.
4. An root key used by a Publisher is not stored securely and the private key is lost. A new key pair is created to continue operations.
5. A cryptographic algorithm is deprecated, or a compliance standard requires a Publisher to upgrade their key strength.

## Approaches
1. Storage and distribution of trust policies from registry

Pros
- Trust policies need not be stored in another location
- Allows in band/automated distribution and update of trust policies to consumers.

Cons
- Automated update of trust policies can be disruptive to Deployers that consume artifacts. Deployers should be able to control when trust policies are updated.
- Only allows Publishers to define the trust policy. It works well when Publisher and Deployer are the same user/organization.
    - This model does not allow Deployers to independently define trust policies. Furthermore, it may be required that Deployer Admins define trusted publishers, and Deployer Operators only specify/configure which artifacts (from repositories) are deployed.

## Recommendation
- Trust policies SHOULD NOT be stored in the registry. Multiple Deployers can consume artifacts from the same repository, and may need to define trust policies independently.
- Trust policies SHOULD be configured, distributed/updated out of band from artifact updates from registry.