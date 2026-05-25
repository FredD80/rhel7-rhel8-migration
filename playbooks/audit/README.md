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
| `cutover_strategies` | Requested production identity transfer strategies, comma-separated | `dns` |

Allowed `cutover_strategies` values:

- `dns`
- `eip`
- `lb`
- `secondary_ip`

DNS is always included. During AWS capture, if the RHEL 7 source instance has an Elastic IP attached, the audit playbook automatically adds `eip` to the effective cutover strategies so both DNS and EIP are represented in the proposed manifest.

CLI example:

```text
ansible-playbook playbooks/audit/discover-migration-manifest.yml \
  -e migration_id=app01-rhel8-20260525 \
  -e rhel7_source_host=app01-rhel7 \
  -e artifact_bucket=my-migration-artifacts \
  -e aws_region=us-east-1 \
  -e cutover_strategies=dns
```
