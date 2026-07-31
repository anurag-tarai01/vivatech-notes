### 1. All Actors

**Admin actors** — stored in `Admin` entity, distinguished by `AdminType` enum:

| Actor             | AdminType       | Level  | Role constant            |
| ----------------- | --------------- | ------ | ------------------------ |
| Super Admin       | `SUPER_ADMIN`   | 1 (HQ) | `ROLE_SUPER_ADMIN`       |
| Network Admin     | `NETWORK_ADMIN` | 1–3    | role-based               |
| Customer Care     | `CUSTOMER_CARE` | 4      | `ROLE_INTERNAL_AGENT`    |
| Third Party Admin | `THIRD_PARTY`   | 4      | `ROLE_THIRD_PARTY_ADMIN` |
| Biller Admin      | `BILLER`        | 4      | `ROLE_BILLER_ADMIN`      |
| API User          | `API_USER`      | 4      | — (machine account)      |

**End-user actors** — stored in `User` entity, distinguished by `UserType` enum:

| Actor             | UserType            | Role constant                 |
| ----------------- | ------------------- | ----------------------------- |
| Subscriber        | `SUBSCRIBER`        | `ROLE_SUBSCRIBER_USER`        |
| Agent             | `AGENT`             | `ROLE_AGENT_USER`             |
| Resale Agent      | `RESALE_AGENT`      | `ROLE_RESALE_AGENT_USER`      |
| Distributor Agent | `DISTRIBUTOR_AGENT` | `ROLE_DISTRIBUTOR_AGENT_USER` |
| Merchant          | `MERCHANT`          | `ROLE_MERCHANT_USER`          |
| Outlet            | `OUTLET`            | sub-entity under Merchant     |

**Status enums** — `UserStatus`: `PENDING`, `ACTIVE`, `BLOCKED`, `SUSPENDED`, `BARRED_AS_SENDER`, `BARRED_AS_RECEIVER`, `REJECTED` / `AdminStatus`: `PENDING`, `ACTIVE`, `BLOCKED`, `SUSPENDED`