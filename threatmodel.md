## Threat model

It is assumed that an attacker may perform one or more the following actions:

1. intercept and alter network traffic
2. compromise some set of weak crypto algorithms which are supported in some legacy cases
3. compromise a repository, including gaining access to use any keys stored on the repository
4. compromise a signing key, for example due to malicious action or accidental disclosure by the key owner
5. compromise a step in the software supply chain.  This can happen in many different ways, such as by gaining access to the server, compromising the software used in the step of the supply chain, passing different software to a subsequent step than what was intended, or causing an operator to make an error in a step.

While it is not always possible to protect against all scenarios, the system should to the extent possible mitigate and/or reduce the damage caused by a successful attack, detect the occurrence of an attack and notify appropriate parties, yet remain usable for parties operating the system.  Furthermore, the system should recover from successful attacks in a way that presents low operational overhead and risk to users.

Attacker Goals:
1. To have a party deploy a malicious artifact under the attacker's control.
2. To have a party deploy an artifact using a digest or tag that does not have a currently valid signature.  For example, a previous version of an artifact with a signature that was revoked by the signer, or an old version associated with a tag.
3. To have a party deploy a different artifact than the one requested, including an older version when the latest is requested.
4. Disrupt the verification of artifact signatures, for example by making the current version of metadata unavailable.
5. Prevent a party from learning about updates to currently installed artifacts.
6. Convince a party to download large amounts of data, such as signatures or metadata, that interfere with the party's system.
7. Enable future attacks of the above types to be carried out more easily.  For example, by causing a party to trust the attacker's key.

## Out of Scope
The following attacks are considered out of scope for Notary v2:
1. Denial of Service (DoS) attacks.
2. Registry validation. A registry may choose to do validation when artifacts are uploaded, but this validation is out of scope of Notary v2.
