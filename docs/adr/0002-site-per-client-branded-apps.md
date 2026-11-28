# One Frappe site per Client, one Branded App per Client

Each **Client** gets its own Frappe site (Frappe multi-site on shared servers) rather than sharing one multi-tenant database, giving hard data isolation and per-Client config with no tenancy code of our own. Each **Client** also gets its own **Branded App** store listing, built from the single Expo codebase, matching how Indian edtech resellers (e.g. Classplus) operate.
