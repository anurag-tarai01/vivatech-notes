### **Problem Description**

During the consumption of `AgentCreatedEvent` messages by the `MFSNotificationApplication` and `MfsReportingApplication` services, the application threw an `org.modelmapper.ConfigurationException`.

The error highlighted that the destination properties `proofOfAddressFile` and `proofOfAddressFileExtension` inside `UserDto` matched multiple source property hierarchies, causing the mapping engine to fail deterministically.

### **Root Cause Analysis**

By default, ModelMapper uses a **Standard/Implicit Matching Strategy**. This engine tokenizes property names based on camelCase boundaries and attempts to loosely pair source tokens to destination tokens.

Inside `UserDto`, we maintain multiple distinct domain entities with highly repetitive field patterns for file uploads:

- `proofOfAddressFile` / `proofOfAddressFileExtension`
    
- `externalAgentProofOfAddressFile` / `externalAgentProofOfAddressFileExtension`
    
- `resellerProofOfAddressFile` / `resellerProofOfAddressFileExtension`
    
- `agentProofOfAddressFile` / `agentProofOfAddressFileExtension`
    

Because the terms `proof`, `address`, `file`, and `extension` overlap across these four distinct field variations, ModelMapper encountered an **ambiguity conflict**. It could not verify if the flat source values were meant for the global, external agent, reseller, or distributor variant of the fields.

### **Solution Implemented**

We transitioned the configuration to **`MatchingStrategies.STRICT`** (or explicitly bypassed the specific ambiguous mappings).

**Why this fixes it:**

1. **Strict Hierarchy Matching:** In `STRICT` mode, tokens must match the exact string sequence and structural depth of the class layout. ModelMapper will no longer attempt to guess or map fields with partial token similarities.
    
2. **Deterministic Processing:** It completely removes ambiguity, ensuring that `proofOfAddressFile` only maps when an exact, uncompromised property path match exists.
    
3. **Fail-Safe Evolution:** As we continue adding specialized user types (e.g., Merchants, Resellers, Distributors) with similar document-handling suffixes, this prevents future regressions across shared DTO conversions.