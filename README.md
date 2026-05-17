# Snowflake Data Sharing Guide

A comprehensive, hands-on guide to every data sharing option in Snowflake — from simple secure shares to native apps and data clean rooms.

## What's Inside

| File | Description |
|------|-------------|
| `01_overview.md` | All 8 sharing options compared — architecture diagram, decision tree, comparison matrix |
| `02_security_governance.md` | RBAC hierarchy, secure views, row access & masking policies, network security, audit, naming conventions, cost model |

### Interactive Notebooks

Each notebook is **self-contained** — includes setup, working examples, and teardown. Run them independently in any order.

| Notebook | Topic |
|----------|-------|
| `notebooks/03_direct_sharing.ipynb` | Direct data sharing (secure shares) |
| `notebooks/04_marketplace_listings.ipynb` | Snowflake Marketplace (public & private listings) |
| `notebooks/05_org_listings.ipynb` | Org listings (internal marketplace) |
| `notebooks/06_declarative_sharing.ipynb` | Declarative sharing (application packages TYPE=DATA) |
| `notebooks/07_native_apps.ipynb` | Native apps framework (data + logic + UI) |
| `notebooks/08_clean_rooms.ipynb` | Data clean rooms (privacy-preserving collaboration) |
| `notebooks/09_replication.ipynb` | Database replication (cross-region, DR) |
| `notebooks/10_security_governance.ipynb` | Security deep dive (RBAC, policies, audit) |

## How to Use

1. **Start with `01_overview.md`** to understand which sharing option fits your use case
2. **Review `02_security_governance.md`** for RBAC patterns and security best practices
3. **Open the relevant notebook** in a Snowflake Workspace or any Jupyter-compatible environment
4. **Run cells top-to-bottom** — each notebook creates its own demo objects and cleans up at the end

### Prerequisites

- A Snowflake account (Enterprise Edition recommended for policy examples)
- `ACCOUNTADMIN` role access (for initial setup cells only — notebooks create least-privilege roles for all data operations)
- Access to `SNOWFLAKE_SAMPLE_DATA` (included with all Snowflake accounts)

### Cross-Account Examples

Some notebooks (direct sharing, replication, org listings) include steps that require a second Snowflake account. These steps are commented out with placeholder account identifiers — uncomment and replace with your target account to run them.

## Conventions

- **SQL:** Uppercase keywords, `snake_case` identifiers
- **RBAC:** Every notebook creates dedicated roles (`share_admin`, `share_monitor`, etc.) and drops them at teardown
- **Security:** All shared objects use secure views — no base tables are exposed directly
