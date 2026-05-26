# Playbooks

This directory follows the minimum playbook layout from the migration specification.

Current implementation status:

- `audit/discover-migration-manifest.yml` is implemented as a non-mutating discovery playbook that produces sanitized audit output and a proposed manifest.
- The remaining phase playbooks are scaffolds and intentionally fail until implemented. This prevents an incomplete workflow from reporting success.
