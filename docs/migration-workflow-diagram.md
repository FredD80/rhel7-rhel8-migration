# Migration Workflow Diagram

```mermaid
flowchart TD
  A["Phase 1: Planning"] --> B["Phase 2: Audit RHEL 7"]
  B --> C["Proposed manifest"]
  C --> D["Review and approve manifest"]
  D --> E["Phase 3: Build RHEL 8"]
  E --> F["Build validation"]
  F --> G["Phase 4a: Setup rsync SSH channel"]
  G --> H["Phase 4b: Selective root sync"]
  H --> I["Phase 5: Pre-cutover validation"]
  I --> J{"Cutover approved?"}
  J -- "No" --> H
  J -- "Yes" --> K["Phase 6: Cutover"]
  K --> L["Move data volumes"]
  L --> M["Transfer production identity"]
  M --> N["Phase 7: Post-cutover validation"]
  N --> O{"Rollback needed?"}
  O -- "Yes" --> P["Phase 8: Rollback"]
  O -- "No" --> Q["Phase 8: Stabilization"]
  Q --> R["Phase 9: Decommission RHEL 7"]
```

## Repository Structure

```text
.
├── docs/
├── group_vars/
├── inventories/
├── lookup_tables/
├── playbooks/
│   ├── audit/
│   │   └── tasks/
│   ├── build/
│   ├── cutover/
│   ├── decommission/
│   ├── migrate/
│   ├── rollback/
│   ├── validate-post/
│   ├── validate-pre/
│   └── validate-shared/
├── roles/
└── schemas/
```
