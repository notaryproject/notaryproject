# Key Revocation

One of the goals of Notary v2 is to build in solutions for key revocation that are easy to use and ensure that users will always use non-compromised keys. This document discusses some potential mechanisms for key revocation.

In existing systems, there are three main approaches to key revocation: automatic revocation through key expiration, key revocation lists, and providing a list of trusted keys. This document discusses some of the benefits and pitfalls of each of these techniques, and how some of these techniques are combined to provide a holistic approach to key revocation.


## Key Expiration

Adding an expiration time to every key allows keys to automatically be revoked after a certain period of time. The expiration time is usually included with the key, and signed by a trusted issuing key, so that it is easy for users to find. This technique does not require any action from the key holder, and ensures that users will have to refresh their trusted keys before those keys expire.

However, if a key is compromised before it expires, key expiration alone does not protect users. An attacker with the compromised key could sign arbitrary images or metadata until the key's expiration time.

Pros:
* No action required from key holder
* Simple: only requires a timestamp

Cons:
* Keys can't be revoked before expiration
* Artifacts must be re-signed after expiration, unless a timestamping service is used.

### Approaches

Developers can have keys with different expiry times.

Expiry time more than or equal to 1 year is generally considered as **long term**, and expiry time less than or equal to 1 day is generally considered as **short term**. Other expiry time ranges falls in to long term or short term depending on various scenarios.

In some scenarios, the developers can use self-signed certificates or even deploy using the public key directly without a third party. When the public key is directly used, it means life-time validity.

#### Long Term

Longer expiry can make the private key exposed for a longer time, which implies larger blast radius. In other words, more artifacts are impacted when a key with longer key duration is rescinded.

Long term keys are generally associated with roots, used infrequently, kept offline when not in use, and need additional protections (e.g. [HSMs](https://en.wikipedia.org/wiki/Hardware_security_module)) against compromise. Thus long term keys SHOULD not be used as signing keys.

#### Short Term

Since the key has short expiry that expires in days or even in minutes, the signer has to generate the key pair and get the public key certified more often by a issuing CA. This requires a robust and secure signing infrastructure that is online and can issue certificates at a higher frequency. Community developers can choose a common trusted public CA to authenticate themselves and certify their short-term ephemeral keys.

* As the key expiry is shorter, timestamping MUST be used where the artifact is expected to be consumed beyond the key expiry.
* Lowers risks of signing key compromise due to lower blast radius, when used along with timestamping.
* Key storage can be simpler as compromised keys have limited blast radius and does not need to be protected for longer periods.

### Recommendation

* Publishers should decide whether long term or short term keys are suitable for their use case based on their internal threat assessment.
* Timestamping SHOULD be used for both long term and short term keys for added protection against compromised keys.
* When artifacts are consumed beyond signing key expiry (more common for short term keys), timestamping MUST be used.
* Deployer MUST be able to specify whether timestamp signature is required, and if expired signatures are allowed.


## Key revocation lists (Deny lists)

Distributing a list of revoked keys allows key holders to notify users quickly if a key is compromised or lost. Users can check against the key revocation list before using a key to verify signatures. This technique is more flexible than key expiration times, and allows for more current information about key security.

However, the user must be able to ensure that the key revocation list is accurate and up to date. If an attacker is able to replay an old revocation list, or show different versions to different registries, the user may continue to trust compromised keys. Therefore the distribution of the key revocation list must allow the user to verify authenticity and timeliness.

Also, for security reasons, keys cannot be removed from a key revocation list, so the list will grow larger and larger over time and may eventually have a noticeable bandwidth impact, although this can be mitigated by combining key revocation lists with keys that expire.

Pros:
* Allows key revocation at any time
* Anyone can add a key to the revocation list

Cons:
* Distribution of the key revocation list needs to be secured and verified for timeliness to prevent replays
* Additional maintenance overhead
* What to do when the revocation list query fails?
* Revocation lists must be synchronized between registries.


## List of trusted keys (Allow lists)

Instead of distributing untrusted keys, this method distributes a list of currently trusted keys. If a key needs to be revoked, it is removed from the list of trusted keys. This technique has the added benefit of ensuring that users have access to the new trusted key as soon as they learn of a revocation.

Similar to revocation lists, the list of trusted keys must be securely distributed and include mechanisms for the user to verify timeliness.

Pros:
* Allows key revocation at any time
* Can be combined with key distribution
* Does not require an additional query

Cons:
* Distribution of trusted key list needs to be secured and verified for timeliness to prevent replays
* Additional maintenance overhead


## Combining explicit and implicit revocation

A final option is to use a combination of the first and third techniques to achieve both implicit and explicit key revocation. By using a hierarchical combination of keys, a trusted root key can delegate signing to various keys that expire. These keys may be revoked before that expiration time by replacing the key listed in the delegation. Additionally, artifacts may be signed by more than one key, allowing automated tooling to provide short lived signatures that verify the signer and artifact have not been revoked. Clients then verify the necessary collection of signatures is found on the artifact.

This method allows signers to have relatively long lived keys, to simplify their workflow and avoid needing to resign the artifacts themselves, while enabling timely revoking of the signing key or a single artifact signature.

For efficiency, a meta-artifact can be created and maintained, containing references to a collection currently signed artifacts. And the short lived signature can be created for this single artifact, rather than every artifact individually. This meta-artifact would need to be updated whenever the collection of artifacts changes and parsed when validating any artifact.

Pros:
* Allows key revocation at any time
* Keys will expire and so will not be used forever
* Individual artifact signatures may be quickly revoked
* Signers do not need to frequently resign all artifacts
* Can be combined with key distribution, so verifiers only need to trust the root key, all delegated keys can be verified against this

Cons:
* Requires maintenance of an automated system to refresh short lived signatures
* A root key compromise requires updating all signers, clients, and signatures on the artifacts
* Updating short lived signatures on a large number of artifacts may encounter scaling challenges and loses some of the caching efficiencies of content addressable storage in registries
