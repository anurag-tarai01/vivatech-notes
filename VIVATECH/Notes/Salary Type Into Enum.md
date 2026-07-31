The payment type was previously represented as free-form strings across entities, DTOs, controllers and services. I introduced a `SalaryType` enum and refactored usages to make payment types type-safe and centralized. This removes string comparisons, prevents typo-related bugs, improves IDE refactoring support, and ensures only valid payment types can be persisted through `@Enumerated(EnumType.STRING)`. I also updated the affected entities and DTOs to use the enum consistently.

That explanation focuses on:

- type safety
- maintainability
- consistency
- reduced bugs