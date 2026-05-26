# Audit Playbook

Entry point:

```text
playbooks/audit/discover-migration-manifest.yml
```

## Mandatory Inputs

Configure these as AAP Job Template survey variables or provide them as CLI extra vars. The playbook does not use `vars_prompt`; AAP surveys are the supported input path.

| Variable | Purpose | Example |
|---|---|---|
| `migration_id` | Names the migration and S3 artifact prefix | `app01-rhel8-20260525` |
| `rhel7_source_host` | Inventory host or group for the RHEL 7 source | `app01-rhel7` |
| `artifact_bucket` | S3 bucket for audit and manifest artifacts | `my-migration-artifacts` |
| `aws_region` | AWS region for S3 and AWS metadata capture | `us-east-1` |
| `cutover_strategies` | Requested production identity transfer strategies, comma-separated | `dns` |

Recommended AAP survey configuration:

| Question | Variable | Type | Required | Default |
|---|---|---|---|---|
| Migration ID | `migration_id` | Text | Yes | none |
| RHEL 7 source host/group | `rhel7_source_host` | Text | Yes | none |
| S3 artifact bucket | `artifact_bucket` | Text | Yes | none |
| AWS region | `aws_region` | Text | Yes | none |
| Cutover strategies | `cutover_strategies` | Multiple choice or text | Yes | `dns` |
| Upload to S3 | `audit_upload_to_s3` | Boolean | No | `true` |

Allowed `cutover_strategies` values:

- `dns`
- `eip`
- `lb`
- `secondary_ip`

DNS is always included. During AWS capture, if the RHEL 7 source instance has an Elastic IP attached, the audit playbook automatically adds `eip` to the effective cutover strategies so both DNS and EIP are represented in the proposed manifest.

## Outputs

The playbook writes two controller-local files and uploads the same files to S3 when `audit_upload_to_s3` is true:

```text
s3://<artifact_bucket>/rhel-migration/<migration_id>/phase-2-audit/sanitized/<timestamp>-sanitized.yml
s3://<artifact_bucket>/rhel-migration/<migration_id>/manifest/proposed/<timestamp>-proposed.yml
```

For local syntax or dry-run development without S3 upload, pass:

```text
-e audit_upload_to_s3=false
```

CLI example:

```text
ansible-playbook playbooks/audit/discover-migration-manifest.yml \
  -e migration_id=app01-rhel8-20260525 \
  -e rhel7_source_host=app01-rhel7 \
  -e artifact_bucket=my-migration-artifacts \
  -e aws_region=us-east-1 \
  -e cutover_strategies=dns
```
