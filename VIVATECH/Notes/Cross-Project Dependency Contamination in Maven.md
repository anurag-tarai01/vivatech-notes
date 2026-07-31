### 📌 Core Concept: Folder Isolation vs. System Registry

When you split your projects into separate directories (`mcash/` and `novapay/`), your operating system isolates them. However, **Maven operates globally at the user-account level.** Both your `G-Pay` and `Nova Pay` codebases utilize matching Maven coordinates (Group ID, Artifact ID, and Version) for their shared internal modules (like backend common libraries, notification DTOs, or reporting aggregates).

### 🔄 The Contamination Mechanism

1. **The Shared Cache (`.m2`):** Every Maven build on your machine resolves and stores dependencies in a single, unified folder: `~/.m2/repository/`.
    
2. **The Overwrite Hook:** Running `mvn clean install` in the `mcash` (G-Pay) workspace compiles its specific code structure and copies the output artifact into that central system directory.
    
3. **The Poisoned Context:** The next time your `novapay` app bootstraps or loads inside your IDE, it looks at the exact same folder coordinates in `~/.m2` to bind the core tracking models. Because the G-Pay build just updated that target pointer, your Nova Pay application pulls the wrong branch's common classes into its runtime environment.
    

### ⚠️ Impact on Microservices

Because your system relies on an event-driven architecture with shared contracts:

- **Payload Deserialization Crashes:** Mismatches occur when your reporting or notification handlers ingest automated events (like `AgentCreatedEvent`). They try to process the payload against an out-of-sync class layout introduced by the other branch's build.
    
- **ModelMapper & Serialization Errors:** Reflection engines and mapping layers fail to align source-to-destination trees cleanly because class definitions mutate depending on whichever workspace executed the last compilation install.