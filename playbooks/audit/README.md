# Audit Playbook

Entry point:

```text
playbooks/audit/discover-migration-manifest.yml
```

## Mandatory Inputs

Configure these as required AAP survey variables or provide them as CLI extra vars.

| Variable | Purpose | Example |
|---|---|---|
| `migration_id` | Names the migration and S3 artifact prefix | `app01-rhel8-20260525` |
| `rhel7_source_host` | Inventory host or group for the RHEL 7 source | `app01-rhel7` |
| `artifact_bucket` | S3 bucket for audit and manifest artifacts | `my-migration-artifacts` |
| `aws_region` | AWS region for S3 and AWS metadata capture | `us-east-1` |
| `cutover_strategy` | Production identity transfer strategy | `dns` |

Allowed `cutover_strategy` values:

- `dns`
- `eip`
- `lb`
- `secondary_ip`

CLI example:

```text
ansible-playbook playbooks/audit/discover-migration-manifest.yml \
  -e migration_id=app01-rhel8-20260525 \
  -e rhel7_source_host=app01-rhel7 \
  -e artifact_bucket=my-migration-artifacts \
  -e aws_region=us-east-1 \
  -e cutover_strategy=dns
```
