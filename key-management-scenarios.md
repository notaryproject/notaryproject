# Notary Key Management Scenarios
In order to allow for end to end signing of artifacts, Notary v2 will require the use of cryptographic keys. These scenarios define how these keys will be generated, distributed, and trusted in Notary v2.

## Scenarios

### Scenario #1: A developer signs a package for the first time
The first time a developer signs a package, they will need to set up a signing key and their local environment.
  1. Generate a developer signing key.
  1. Create a local environment that delegates to the developer key for testing.
  1. The developer signs local content.

### Scenario #2: A developer is given permission to push artifacts to a registry (for the first time)
The first time a developer pushes an artifact to a registry, they need to establish that their keys are trusted for the artifacts they push.
  1. Add the developer key to delegating party on the registry. There may be a single delegating party on a registry, or multiple that are trusted for different artifacts.

### Scenario #3: A developer updates their signing key
In the event of a compromised or lost key, a developer may need to switch to a new signing key.
  1. The developer will generate a new signing key.
  1. The new developer key will need to be added to the delegating party on the registry.
  1. The compromised developer key should no longer be trusted.
