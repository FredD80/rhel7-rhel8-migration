# RHEL 7 to RHEL 8 AWS Migration - End-to-End Specification

## 1. Purpose

This document defines the end-to-end process for migrating RHEL 7 hosts to RHEL 8 in AWS using Ansible Automation Platform (AAP). It is written for a FedRAMP-controlled environment where auditability, repeatability, artifact retention, and least privilege are required.

This specification is the controlling document for the migration automation repository. It replaces ad hoc manual steps with a phase-based process tied together by a `migration_id`.

## 2. Core Decisions

The following decisions are mandatory for this migration model.

1. RHEL 8 is built side-by-side. No in-place upgrade is performed.
2. Data volumes are detached from RHEL 7 and attached to RHEL 8 during cutover. Data-volume content is not rsynced.
3. Root-volume content is migrated selectively. Do not bulk copy `/`, `/etc`, `/usr`, `/var/lib/rpm`, `/boot`, `/lib`, `/lib64`, device files, or package-managed operating system content.
4. All host-specific migration data lives in S3. Bitbucket contains automation code only.
5. The approved manifest in S3 is the migration contract. Playbooks do not consume unapproved audit output.
6. Terraform is preferred for provisioning, but manual provisioning is allowed when the manual build checklist is completed and the validation playbook passes.
7. Every workflow is short-lived and independently launched. No AAP workflow waits across the multi-week parallel phase.
8. Cutover is gated by a recent passing pre-cutover validation artifact and the exact git commit SHA that produced it.
9. Rollback semantics must be approved by the application owner before technical work begins.
10. DNS cutover is the default production identity transfer strategy. EIP reassociation, load balancer target swap, or secondary private IP reassignment require an explicit approved exception in the manifest.
11. Primary ENI movement is not a supported cutover strategy.

## 3. Control Plane

### 3.1 Migration ID

Every migration receives a unique `migration_id`. The ID is used in:

- S3 artifact paths
- AAP extra vars
- AWS tags
- snapshot tags
- temporary SSH key comments
- security group rule descriptions
- change ticket references
- validation reports

Recommended format:

```text
<app-or-hostname>-rhel8-<yyyymmdd>
```

### 3.2 Source Control

Bitbucket contains automation only:

- playbooks
- roles
- schemas
- lookup tables
- inventories
- group vars
- documentation

Bitbucket must not contain:

- raw audit captures
- host-specific manifests
- approved manifests
- SSH keys
- secrets
- cert private keys
- package repository credentials
- validation artifacts
- cutover logs

### 3.3 S3 Artifact Store

All migration data is stored in S3 under:

```text
s3://<bucket>/rhel-migration/<migration_id>/
```

Required prefixes:

```text
phase-1-planning/
phase-2-audit/
manifest/proposed/
manifest/approved/
phase-3-build/
phase-4-migrate/
validate-pre/
phase-6-cutover/
validate-post/
phase-8-stabilization/
phase-8-rollback/
phase-9-decommission/
```

The bucket must have:

- SSE-KMS using an approved customer-managed KMS key
- versioning enabled
- Object Lock retention if required by the authorization package or organizational policy
- CloudTrail data events enabled for object-level access
- bucket policy requiring TLS
- bucket policy requiring approved KMS encryption
- lifecycle and retention rules aligned with the FedRAMP package and organizational policy

CloudTrail logs should be retained separately from the migration artifact bucket.

### 3.4 Approved Manifest

The approved manifest is stored at:

```text
s3://<bucket>/rhel-migration/<migration_id>/manifest/approved/approved.yml
```

The upload is the approval event. IAM controls who may write to the approved prefix. CloudTrail records who uploaded it and when.

Any job that consumes the approved manifest must record:

- S3 bucket
- S3 key
- S3 version ID
- SHA-256 checksum
- approver identity if available from object metadata or external ticket
- download timestamp

The pre-cutover validation artifact records the exact approved manifest version. Cutover must fail if the manifest version or checksum differs from the one validated.

## 4. Required AAP Job Metadata

Every AAP job that produces or changes migration state must write a metadata block to its S3 artifact.

Required fields:

```yaml
metadata:
  migration_id: ""
  phase: ""
  aap_job_id: ""
  aap_workflow_id: ""
  git_repository_url: ""
  git_branch: ""
  git_commit_sha: ""
  execution_environment_image: ""
  execution_environment_digest: ""
  ansible_version: ""
  ansible_collection_versions: {}
  started_at: ""
  completed_at: ""
  result: ""
```

The git commit SHA and execution environment digest are reproducibility anchors.

## 5. Manifest Requirements

The manifest is the approved machine-readable contract for the migration.

Required top-level fields:

```yaml
manifest_version: 1
migration_id: ""
change_ticket: ""
source:
  instance_id: ""
  hostname: ""
  fqdn: ""
  private_ip: ""
  availability_zone: ""
  vpc_id: ""
  subnet_id: ""
  security_group_ids: []
  iam_instance_profile: ""
target:
  provisioning_method: "manual" # manual or terraform
  instance_id: ""
  temporary_hostname: ""
  private_ip: ""
  availability_zone: ""
  vpc_id: ""
  subnet_id: ""
  security_group_ids: []
  iam_instance_profile: ""
compliance:
  fedramp_boundary: ""
  fips_required: false
  crypto_policy: "DEFAULT" # DEFAULT or FIPS
  selinux_required_state: "enforcing"
  openscap_profile: ""
  tailoring_file: ""
cutover:
  strategy: "dns" # dns is default; eip, lb, or secondary_ip require approved exception
  dns:
    hosted_zone_id: ""
    record_name: ""
    record_type: "A"
    source_value: ""
    target_value: ""
    pre_cutover_ttl: 60
    restore_ttl: 300
  ttl_restore_required: true
rollback:
  window_hours: 0
  rpo_statement: ""
  owner_approval_reference: ""
root_sync_paths: []
data_volumes: []
users_and_groups: []
ssh: {}
repositories: []
services: []
dependencies: []
approvals: {}
```

### 5.1 Root Sync Path Contract

Each root sync path must be explicitly approved.

Example:

```yaml
root_sync_paths:
  - path: /opt/myapp
    owner_service: myapp.service
    one_file_system: true
    delete_allowed: false
    preserve_acls: true
    preserve_xattrs: true
    preserve_hardlinks: true
    preserve_numeric_ids: true
    excludes:
      - "*.tmp"
      - "cache/"
    validation:
      file_count_parity: true
      size_parity: true
      sha256_spotcheck:
        - /opt/myapp/conf/app.conf
    selinux:
      restorecon_after_sync: true
      expected_contexts: []
```

The sync playbook must default to deny. If a path is not in `root_sync_paths`, it is not copied.

### 5.2 Data Volume Contract

Each data volume must record AWS and Linux identity.

Example:

```yaml
data_volumes:
  - volume_id: vol-00000000000000000
    source_device_name: /dev/sdf
    linux_device_hint: /dev/nvme1n1
    by_id_path: /dev/disk/by-id/nvme-Amazon_Elastic_Block_Store_vol00000000000000000
    filesystem: xfs
    mount_point: /data
    lvm:
      present: false
      volume_group: ""
      logical_volumes: []
    encrypted: true
    kms_key_id: ""
    volume_type: gp3
    size_gib: 0
    iops: 0
    throughput: 0
    multi_attach: false
    delete_on_termination: false
    cutover_handling: clean_unmount # clean_unmount, app_dump, reverse_replication_required
```

Btrfs is a blocker for simple detach and attach. If Btrfs is found, a separate migration plan is required.

### 5.3 SSH Contract

The audit identifies SSH material. Migration of credential material requires explicit approval.

Example:

```yaml
ssh:
  host_key_strategy: generate_new # generate_new or preserve_existing
  source_host_key_fingerprints:
    - algorithm: ssh-ed25519
      fingerprint: SHA256:example
  migrate_sshd_config: true
  users:
    - name: appuser
      uid: 1005
      gid: 1005
      migrate_authorized_keys: true
      authorized_key_fingerprints:
        - SHA256:example
      migrate_client_config: false
      migrate_private_keys: false
```

Default behavior:

- migrate approved `authorized_keys`
- migrate required service account SSH client config only when approved
- migrate required `known_hosts` entries only when approved
- do not migrate user private keys by default
- do not migrate root private keys by default
- do not migrate SSH host private keys by default

If host keys must be preserved to avoid client known-host failures, this is credential migration and must be approved, logged, and protected as sensitive data.

### 5.4 Repository and GPG Contract

Do not trust captured RHEL 7 repository data by default. The RHEL 8 build must compare repo URLs and GPG fingerprints against an approved allowlist.

Example:

```yaml
repositories:
  - name: rhel-8-for-x86_64-baseos-rpms
    enabled: true
    baseurl: ""
    approved: true
    gpg_key_fingerprints:
      - ""
  - name: internal-vendor-rhel8
    enabled: true
    baseurl: ""
    approved: true
    gpg_key_fingerprints:
      - ""
```

Captured RHEL 7 repos are input to review. They are not automatically trusted.

### 5.5 User and Group Contract

Local application users and groups should preserve UID/GID where required. Package-managed system users should not be blindly recreated.

The build playbook must detect:

- UID conflicts
- GID conflicts
- existing RHEL 8 package-owned accounts
- accounts outside the approved allowlist

Conflicts fail the build until resolved in the manifest.

## 6. Security Group and Network Rules

Same VPC and same subnet are not sufficient. Security groups still control instance-to-instance traffic.

The rsync model is pull-mode:

```text
RHEL 8 -> RHEL 7 TCP/22
```

Before Phase 4, validate:

- RHEL 8 outbound allows TCP/22 to RHEL 7 or to the bastion
- RHEL 7 inbound allows TCP/22 from RHEL 8, the RHEL 8 security group, or the bastion
- NACLs allow the traffic and ephemeral return traffic
- host firewalls allow SSH
- `sshd` is listening on RHEL 7

If direct RHEL 8 to RHEL 7 SSH is not approved, configure RHEL 8 to use the bastion with `ProxyJump`.

Any temporary security group rules must include:

- `migration_id`
- change ticket
- expiration or cleanup phase
- narrow source and destination
- narrow port range

Phase 9 must remove temporary SG rules.

## 7. Ephemeral Rsync SSH Channel

AAP has machine credentials to each host, but rsync requires RHEL 8 to SSH to RHEL 7 directly or through a bastion.

Channel setup creates a per-migration key on RHEL 8:

```text
/root/.ssh/migration_<migration_id>
```

Required controls:

- ed25519 keypair
- file mode `0600`
- key exists only for this migration
- public key added to RHEL 7 root `authorized_keys` with a `migration_id` comment
- RHEL 7 host key fingerprint captured during audit
- setup compares `ssh-keyscan` result against the approved fingerprint before writing `known_hosts`
- no `StrictHostKeyChecking=no`
- S3 evidence records key fingerprint, host fingerprint, timestamp, and job metadata

The authorized key on RHEL 7 should be restricted with options similar to:

```text
from="<rhel8-private-ip-or-cidr>",command="/usr/local/sbin/migration-rsync-wrapper <migration_id>",restrict,no-pty,no-agent-forwarding,no-X11-forwarding,no-port-forwarding ssh-ed25519 <public-key> migration-<migration_id>
```

The forced-command wrapper must allow only approved rsync reads from manifest-approved paths.

Teardown removes:

- RHEL 7 `authorized_keys` entry
- RHEL 8 private and public migration keys
- RHEL 8 known_hosts entry if it was created only for migration

## 8. Workflow Overview

| Phase | Workflow | Trigger | Frequency |
|---|---|---|---|
| 1 | Planning | Manual/process | Once |
| 2 | `rhel-migration-audit` | Manual | Once per host, rerun if source changes materially |
| 3 | `rhel-migration-build` | Manual after approved manifest | Once |
| 4a | `rhel-migration-setup` | Manual after build | Once |
| 4b | `rhel-migration-data-sync` | Schedule after setup | Recurring during parallel phase |
| 5 | `rhel-migration-validate-pre` | Manual | Repeated |
| 6 | `rhel-migration-cutover` | Manual after CAB | Once |
| 7 | `rhel-migration-validate-post` | Auto from cutover, manual rerun allowed | Once or more |
| 8 | `rhel-migration-healthcheck` | Schedule | Daily during rollback window |
| 8 | `rhel-migration-rollback` | Manual | Only if invoked |
| 9 | `rhel-migration-decommission` | Manual after rollback window | Once |

Approval gates are outside long-running workflows:

- audit reviewed and manifest approved
- build complete
- setup complete
- data-sync schedule enabled
- pre-cutover validation passed and CAB approved
- rollback window closed

## 9. Phase 0 - Platform Readiness

This phase prepares the environment before host migrations begin.

Required:

1. AAP is configured and can reach RHEL 7 and RHEL 8 hosts.
2. AAP project syncs the automation repo from Bitbucket.
3. AAP project uses update-on-launch or equivalent fresh SCM sync behavior.
4. AAP execution environment includes required collections and Python modules.
5. Execution environment image digest is recorded.
6. S3 artifact bucket exists with SSE-KMS, versioning, retention, and CloudTrail data events.
7. KMS key policy allows required roles to encrypt, decrypt, describe, create grants, and re-encrypt as needed.
8. IAM roles exist for audit, approval, operation, and AAP execution.
9. Security team has approved the temporary migration network pattern.
10. CloudTrail is enabled for relevant AWS API actions.
11. AWS services in use are confirmed in scope for the applicable FedRAMP program.

If Terraform is not used, document the manual provisioning procedure and require validation evidence from AAP.

## 10. Phase 1 - Planning

Purpose: lock down scope, approvals, rollback semantics, and network decisions.

Steps:

1. Assign `migration_id`.
2. Identify source instance ID, hostname, AZ, VPC, subnet, IAM role, security groups, and KMS keys.
3. Confirm target RHEL 8 must be created in the same AZ.
4. Select provisioning method: manual or Terraform.
5. Select DNS cutover unless an exception is approved for EIP, LB target swap, or secondary private IP.
6. Confirm primary ENI move is not being used.
7. Decide FIPS requirement before build.
8. If FIPS is required, confirm the build process enables FIPS before workload configuration and key generation.
9. Select crypto policy: `DEFAULT` or `FIPS`.
10. Confirm SELinux enforcing requirement.
11. Identify data volumes and whether each can be cleanly unmounted or needs app-native dump, queue drain, or reverse replication.
12. Confirm instance type supports required EBS volume count, throughput, IOPS, and ENI/IP needs.
13. Confirm KMS access requirements.
14. Map downstream dependencies, ports, callers, scheduled jobs, and service owners.
15. Decide rollback RPO and point of no return.
16. Capture application owner sign-off for rollback semantics.
17. Create or update change ticket.
18. Schedule maintenance window.
19. Define success criteria.
20. Define rollback triggers.
21. Lower DNS TTL to 60-300 seconds at least 24 hours before cutover.
22. Record DNS TTL restoration requirement for Phase 9.
23. Confirm RHEL 7 to RHEL 8 proof of concept in the same AWS account or partition, KMS model, STIG baseline, RHSM/repo model, agents, and AAP execution environment.

Outputs:

- change ticket
- per-host planning record
- rollback RPO approval
- initial manifest skeleton
- stakeholder notification log

## 11. Phase 2 - Audit

Purpose: capture source state into S3 and produce a proposed manifest for review.

The audit playbook runs on RHEL 7. It must not change source state except for harmless temporary commands needed for discovery.

### 11.1 OS Captures

Capture:

1. hostname, FQDN, release, kernel, uptime, boot parameters
2. package inventory from `rpm -qa`
3. yum repository files, yum config, enabled and disabled repos, installed package to repo mapping, yum plugins, RHSM state
4. GPG public key fingerprints from RPM DB and `/etc/pki/rpm-gpg/`
5. systemd enabled and running units
6. custom unit files in `/etc/systemd/system/`
7. users and groups with UID/GID
8. sudoers and `/etc/sudoers.d/`
9. SSH daemon config, daemon config fragments, public host key fingerprints, per-user authorized key fingerprints
10. per-user SSH client config and known_hosts for service accounts, if applicable
11. filesystem layout, mounts, LVM topology, fstab, disk usage
12. network config, routes, DNS, chrony/ntp, firewalld zones, firewalld direct rules, iptables rules
13. SELinux mode, policy modules, booleans, file contexts
14. cron, systemd timers, at jobs, user crontabs
15. sysctl, kernel modules, modprobe config
16. certificates and keystores, including CA trust anchors
17. listening ports and established connections
18. application inventory, versions, config paths, data paths, log paths
19. RHEL 7 to RHEL 8 compatibility checks: Python shebangs, ntp service, MTA, authconfig/SSSD, FIPS state, database versions, Btrfs presence
20. performance baseline from available `sar` data

Do not copy SSH private keys, host private keys, cert private keys, keytabs, or secrets into sanitized artifacts.

### 11.2 AWS Captures

Capture:

1. source instance ID, AZ, instance type, AMI, launch time
2. IAM instance profile
3. security groups and rules
4. full tag set
5. ENI IDs, primary IP, secondary IPs, EIP association
6. attached EBS volumes with size, type, IOPS, throughput, encryption, KMS key, attachment name, delete-on-termination flag
7. LVM mapping to EBS volumes
8. AWS Backup associations
9. CloudWatch agent and SSM registration state
10. relevant Route 53 records, EIP allocations, or load balancer target group memberships depending on cutover strategy

### 11.3 Audit Outputs

Write to S3:

```text
phase-2-audit/raw/<timestamp>-raw.yml
phase-2-audit/sanitized/<timestamp>-sanitized.yml
manifest/proposed/<timestamp>-proposed.yml
```

Raw artifacts may include sensitive operational details and must have restricted access.

The proposed manifest is reviewed manually. The reviewer edits it into an approved manifest and uploads it to:

```text
manifest/approved/approved.yml
```

## 12. Phase 3 - Build

Purpose: create a clean RHEL 8 host that matches the approved target baseline.

Provisioning may be manual or Terraform. If manual, the operator must record the build facts in the manifest and run build validation before continuing.

### 12.1 Manual Provisioning Checklist

If the target is created manually, record:

- operator
- timestamp
- change ticket
- source instance ID
- target instance ID
- AMI ID
- RHEL 8 minor version
- AZ
- VPC
- subnet
- instance type
- IAM instance profile
- security groups
- KMS key for root volume
- root volume size, type, IOPS, throughput
- tags
- temporary hostname
- temporary private IP

### 12.2 Build Steps

1. Provision target RHEL 8 in the same AZ.
2. Attach approved IAM instance profile.
3. Attach approved security groups.
4. Encrypt root volume with approved KMS key.
5. Apply tags from source plus migration tags.
6. If FIPS is required, enable it before workload configuration and before generating or installing long-lived keys.
7. Apply RHEL 8 STIG or tailored OpenSCAP baseline.
8. Set SELinux enforcing.
9. Configure repositories from the approved allowlist.
10. Register with subscription-manager if applicable.
11. Confirm `dnf repolist enabled` matches the approved manifest.
12. Enable required module streams.
13. Install approved package set.
14. Create approved users and groups with exact UID/GID only after collision checks.
15. Configure sudoers.
16. Configure SSH daemon policy.
17. Apply approved authorized keys and service account SSH client config.
18. Generate new SSH host keys unless preservation is explicitly approved.
19. Configure NetworkManager using RHEL 8-supported format.
20. Configure firewalld using zones, rich rules, or policies. Do not blindly paste raw RHEL 7 iptables rules.
21. Configure sysctl and modprobe settings.
22. Configure chrony.
23. Configure MTA, preferably postfix.
24. Configure authselect and faillock.
25. Apply crypto policy.
26. Install certificates and keystores. Rebuild Java trust stores if applicable.
27. Install New Relic, Splunk, and CrowdStrike using existing AAP templates.
28. Configure log shipping and journald persistence if needed.
29. Confirm KMS access by testing a throwaway encrypted EBS volume, then delete the test volume.
30. Confirm AAP inventory contains the target host.
31. Run build smoke tests.

### 12.3 Build Validation

The `manual-build-validate.yml` or build validation playbook verifies:

- target is RHEL 8 expected minor version
- target is in same AZ as source
- target has expected instance type
- root volume is encrypted with approved KMS key
- IAM profile is correct
- security groups are correct
- SELinux is enforcing
- FIPS and crypto policy match manifest
- approved repos are enabled
- unauthorized repos are disabled
- required packages and agents are installed
- users and groups match approved manifest
- SSH daemon policy matches approved manifest
- AAP can connect
- RHEL 8 can reach RHEL 7 on the approved rsync path
- S3 evidence upload works

Outputs:

```text
phase-3-build/<timestamp>-build-validation.yml
```

Approval gate: build complete, proceed to setup.

## 13. Phase 4 - Setup and Migrate

Purpose: prepare the rsync channel, run selective root sync, and maintain data volume rollback artifacts.

### 13.1 Setup Workflow

The one-time setup workflow runs:

1. load approved manifest from S3
2. validate manifest schema
3. record manifest version ID and checksum
4. verify source paths exist on RHEL 7
5. verify target paths are allowed and safe
6. verify data volumes exist in AWS
7. verify network path for RHEL 8 to RHEL 7 SSH
8. verify security group and NACL assumptions
9. configure ephemeral rsync SSH channel
10. upload setup evidence to S3

Outputs:

```text
phase-4-migrate/<timestamp>-setup.yml
```

Approval gate: enable recurring data-sync schedule.

### 13.2 Root Sync

The data-sync workflow runs on schedule during the parallel phase.

Root sync rules:

- pull from RHEL 8
- use the ephemeral migration key
- consume only approved manifest paths
- use `--numeric-ids`
- preserve approved attributes
- use `--one-file-system`
- use `--delete` only where `delete_allowed: true`
- never sync package-managed OS paths
- log transferred bytes, changed files, warnings, and errors

Representative rsync options:

```text
rsync -aAXH --numeric-ids --one-file-system
```

Use `-S` only where sparse files are expected and tested.

After sync:

1. run `restorecon` on synced trees where configured
2. check ownership and UID/GID mapping
3. check ACLs and xattrs
4. run `getcap -r` comparison for file capabilities where needed
5. upload sync report to S3

Outputs:

```text
phase-4-migrate/rsync/<timestamp>-rsync-report.yml
```

### 13.3 Data Volume Snapshots

During the parallel phase:

1. create baseline snapshots of each data volume
2. tag snapshots with `migration_id`
3. schedule recurring snapshots if required
4. record snapshot IDs and volume metadata

These snapshots are crash-consistent unless application quiescing is performed.

Outputs:

```text
phase-4-migrate/snapshots/<timestamp>-snapshot-report.yml
```

## 14. Phase 5 - Pre-Cutover Validation

Purpose: prove the RHEL 8 host is ready before production identity or data volumes move.

This workflow is run repeatedly during the parallel phase. The final passing run must be within the approved freshness window, default four hours.

Steps:

1. load approved manifest from S3
2. record manifest version ID and checksum
3. run final delta rsync
4. verify no unauthorized sync paths were used
5. run `rsync -avnc --delete` dry-run where delete is approved and applicable
6. validate file counts, sizes, and checksums for critical files
7. validate ownership, ACLs, xattrs, modes, setuid/setgid bits, and file capabilities
8. run `restorecon -nvR` and confirm no unexpected context changes
9. start services on localhost, alternate port, or staging binding only
10. validate logs and service readiness
11. check for recent SELinux AVC denials
12. validate outbound connectivity to dependencies
13. validate agent check-in under staging identity
14. run OpenSCAP against approved tailored profile
15. verify TLS and SSH client compatibility with selected crypto policy
16. verify AWS permissions expected from the instance
17. verify KMS access
18. verify cutover-strategy-specific dry-run on approved POC or non-production target
19. verify data-volume mapping using `/dev/disk/by-id`, not `/dev/sdX`
20. verify EBS volume metadata matches manifest
21. compile validation report

Do not use broad SELinux permissive mode as the default. Prefer targeted staging analysis, per-domain permissive where approved, or documented waiver and remediation.

Outputs:

```text
validate-pre/<timestamp>-report.yml
```

The report must include:

- approved manifest S3 version ID
- approved manifest checksum
- git commit SHA
- execution environment digest
- validation result
- CAB reference if approved for cutover

Approval gate: CAB go/no-go.

## 15. Phase 6 - Cutover

Purpose: move production service to RHEL 8 in the maintenance window.

The cutover workflow is launched manually after CAB approval. It does not rely on a long-running parent workflow.

### 15.1 Precondition Check

The first workflow node must verify:

1. recent passing pre-cutover validation artifact exists in S3
2. validation artifact is within freshness window
3. cutover git commit SHA equals validation git commit SHA unless CAB approved a newer commit
4. cutover execution environment digest equals validation digest unless CAB approved a newer digest
5. approved manifest S3 version ID equals validation manifest version ID
6. approved manifest checksum equals validation manifest checksum
7. recurring rsync schedule is disabled or paused
8. maintenance window is active

Fail if any check fails.

### 15.2 Cutover Sequence

1. Announce maintenance start.
2. Put monitoring and alerting into maintenance mode.
3. Stop application services on RHEL 7 in dependency order.
4. Run final root-content rsync delta.
5. Run application-aware final dumps, queue drains, or exports if required.
6. Apply final dumps to RHEL 8 if produced.
7. Confirm no final rsync errors.
8. Unmount data volumes cleanly on RHEL 7.
9. Confirm no busy files remain.
10. Flush filesystem state.
11. Take final snapshot of each data volume.
12. Tag final snapshots with `migration_id` and `cutover-final=true`.
13. Stop RHEL 7 and preserve it for the rollback window.
14. Detach data volumes from RHEL 7.
15. Attach data volumes to RHEL 8.
16. On RHEL 8, discover devices by `/dev/disk/by-id`.
17. If LVM is used, run `vgscan` and `vgchange -ay`.
18. Use `vgimport` only if the volume group was explicitly exported.
19. Use `vgimportclone` only for cloned or snapshot-restored VGs with duplicate UUIDs.
20. Mount data volumes by UUID according to approved fstab.
21. Verify mounts, filesystems, sizes, and SELinux contexts.
22. Set RHEL 8 hostname to production hostname.
23. Update agent identities where required.
24. Start application services on RHEL 8 in dependency order.
25. Run immediate smoke checks.
26. Execute production identity transfer based on `cutover.strategy`.
27. Coordinate firewall or security group updates that reference the old IP.
28. Re-enable monitoring.
29. Launch post-cutover validation.

### 15.3 Production Identity Transfer

DNS:

1. confirm the manifest contains hosted zone ID, record name, record type, source value, target value, cutover TTL, and restore TTL
2. update A or AAAA record from the RHEL 7 private IP to the RHEL 8 private IP
3. verify the authoritative Route 53 value
4. verify resolution from representative downstream consumers
5. record old value, new value, TTL, hosted zone ID, change ID, and timestamps
6. leave TTL restoration for Phase 9 unless the migration plan requires earlier restoration

EIP:

1. disassociate EIP from RHEL 7
2. associate EIP with RHEL 8
3. verify reachability

Load balancer:

1. register RHEL 8 target
2. wait for healthy status
3. deregister RHEL 7 target
4. verify traffic

Secondary private IP:

1. unassign secondary private IP from RHEL 7 ENI
2. assign secondary private IP to RHEL 8 ENI
3. verify OS binding and route behavior
4. coordinate neighbor cache or ARP refresh if applicable

Primary ENI move is not used.

Outputs:

```text
phase-6-cutover/<timestamp>-cutover-log.yml
```

## 16. Phase 7 - Post-Cutover Validation

Purpose: confirm production behavior under real traffic.

Steps:

1. run functional smoke tests
2. test production endpoints
3. validate downstream consumer connectivity
4. compare error rate against baseline at T+15, T+30, and T+60 minutes
5. compare latency against baseline
6. confirm agents checking in under production identity
7. confirm Splunk indexing under correct host identity
8. run OpenSCAP against approved tailored baseline
9. run application regression tests if available
10. verify mounted data volumes
11. run `xfs_info` or filesystem-specific equivalent
12. verify `df` sizes match expected values
13. check SELinux denials
14. collect service status and logs
15. sign off success criteria

Outputs:

```text
validate-post/<timestamp>-report.yml
```

## 17. Phase 8 - Stabilization and Rollback

Purpose: keep rollback viable while the new host proves stable.

Stabilization steps:

1. keep RHEL 7 stopped, not terminated
2. preserve final cutover snapshots
3. run daily health checks on RHEL 8
4. track anomalies
5. hold stakeholder check-ins at T+24 hours, T+72 hours, and T+1 week or as approved
6. decide whether to close or extend rollback window

### 17.1 Rollback Rules

Rollback is not automatic. It requires incident or change approval according to the rollback triggers.

Before rollback:

1. stop or quiesce RHEL 8 services
2. snapshot RHEL 8 root volume and attached data volumes for forensic preservation
3. record the exact reason for rollback
4. confirm application owner accepts the approved RPO impact

### 17.2 Rollback Sequence

1. Put monitoring into maintenance mode.
2. Stop RHEL 8 application services.
3. Snapshot RHEL 8 root and data volumes.
4. Stop RHEL 8 if required.
5. Detach data volumes from RHEL 8.
6. Restore final cutover snapshots to fresh volumes.
7. Attach restored volumes to RHEL 7.
8. Mount volumes on RHEL 7.
9. Start RHEL 7.
10. Start services on RHEL 7.
11. Restore production identity based on original cutover strategy.
12. Validate service health.
13. Capture rollback evidence.
14. Update incident and change records.

Rollback identity transfer must branch by strategy:

- DNS: point DNS back to RHEL 7 private IP
- EIP: move EIP back to RHEL 7
- LB: re-register RHEL 7 and deregister RHEL 8 after health checks
- secondary IP: move secondary private IP back to RHEL 7

Do not attempt primary ENI movement.

Outputs:

```text
phase-8-rollback/<timestamp>-rollback-report.yml
```

## 18. Phase 9 - Decommission

Purpose: remove RHEL 7 and temporary migration access after rollback window closes.

Steps:

1. Confirm rollback window closed and owner approval received.
2. Capture final RHEL 7 disk image or backup if retention policy requires.
3. Tear down ephemeral rsync SSH channel.
4. Confirm migration key removed from RHEL 7 `authorized_keys`.
5. Confirm migration keypair deleted from RHEL 8.
6. Disable and delete recurring AAP schedules for the migration.
7. Remove temporary security group rules.
8. Remove temporary KMS grants if created.
9. Delete throwaway validation EBS volumes.
10. Delete or retain snapshots according to retention policy.
11. Deregister RHEL 7 agents from consoles.
12. Remove RHEL 7 from AAP inventory.
13. Remove RHEL 7 from monitoring.
14. Terminate RHEL 7 through the approved provisioning method or manual API.
15. Release orphaned ENIs, EBS volumes, temporary IPs, and DNS records.
16. Restore DNS TTL if it was lowered.
17. Update CMDB.
18. Update security boundary documentation.
19. Close change ticket.
20. Write lessons learned.

Outputs:

```text
phase-9-decommission/<timestamp>-decommission-report.yml
```

Lessons learned may be committed to the repo only if it contains no host-specific sensitive data.

## 19. Validation Playbook Layout

Recommended structure:

```text
playbooks/
  audit/
    discover-migration-manifest.yml
  build/
    rhel8-build.yml
    manual-build-validate.yml
  migrate/
    load-validate-manifest.yml
    setup-rsync-channel.yml
    rsync-root-content.yml
    snapshot-volumes.yml
    teardown-rsync-channel.yml
  validate-pre/
    precutover.yml
    data-completeness.yml
    attribute-preservation.yml
    service-readiness.yml
    aws-readiness.yml
  validate-post/
    postcutover.yml
    smoke-tests.yml
    consumer-connectivity.yml
    error-rate.yml
    volume-health.yml
  validate-shared/
    agent-checkin.yml
    outbound-connectivity.yml
    openscap-scan.yml
    performance-baseline.yml
  cutover/
    01-precondition-check.yml
    02-maintenance-mode.yml
    03-stop-source-services.yml
    04-final-sync.yml
    05-data-volume-cutover.yml
    06-start-target-services.yml
    07-production-identity-transfer.yml
    08-launch-post-validation.yml
  rollback/
    rollback.yml
  decommission/
    decommission.yml
```

## 20. FedRAMP Evidence Map

| Evidence | Source | S3 location |
|---|---|---|
| Change approval | Change system export or link | `phase-1-planning/` |
| Rollback RPO approval | Application owner approval | `phase-1-planning/` |
| Source baseline | Audit playbook | `phase-2-audit/` |
| Approved manifest | Reviewer upload | `manifest/approved/` |
| Build baseline | Build validation | `phase-3-build/` |
| FIPS and crypto policy | Planning and build validation | `phase-1-planning/`, `phase-3-build/` |
| SELinux enforcing status | Build, pre, post validation | `phase-3-build/`, `validate-pre/`, `validate-post/` |
| OpenSCAP results | Validation playbooks | `validate-pre/`, `validate-post/` |
| Tailoring and waivers | Validation artifact | `validate-pre/`, `validate-post/` |
| SSH channel lifecycle | Setup and teardown | `phase-4-migrate/`, `phase-9-decommission/` |
| Root sync logs | Rsync playbook | `phase-4-migrate/` |
| Snapshot inventory | Snapshot playbook and cutover | `phase-4-migrate/`, `phase-6-cutover/` |
| Cutover log | Cutover workflow | `phase-6-cutover/` |
| Post-cutover validation | Validation workflow | `validate-post/` |
| Rollback evidence | Rollback workflow | `phase-8-rollback/` |
| Decommission evidence | Decommission workflow | `phase-9-decommission/` |
| CloudTrail evidence | CloudTrail | linked from artifacts |
| KMS usage evidence | CloudTrail | linked from artifacts |
| S3 access evidence | CloudTrail data events | CloudTrail bucket |

## 21. Hard Blockers

The migration must stop if any of the following are true:

1. No approved manifest exists in S3.
2. Manifest version or checksum cannot be verified.
3. RHEL 8 target is not in the same AZ when data volumes are being moved.
4. Btrfs is present on a data volume and no alternate plan exists.
5. FIPS is required but was not enabled before workload configuration and key generation.
6. Required KMS permissions are missing.
7. Required security group changes are not approved.
8. RHEL 8 cannot reach RHEL 7 over the approved rsync SSH path.
9. RHEL 7 SSH host key fingerprint does not match the approved audit fingerprint.
10. Repo or GPG fingerprint is not approved.
11. UID/GID collisions are unresolved.
12. Pre-cutover validation is stale or failed.
13. Cutover job commit SHA does not match validation commit SHA and CAB has not approved the difference.
14. Data-volume final snapshot fails.
15. Application owner has not approved rollback RPO semantics.

## 22. Implementation Order

Recommended build order:

1. S3/KMS/IAM/CloudTrail artifact foundation
2. manifest schema and validation
3. audit playbook
4. manual build validation playbook
5. rsync channel setup with host fingerprint verification
6. forced-command rsync wrapper
7. root sync playbook
8. snapshot playbook
9. pre-cutover validation
10. cutover workflow
11. post-cutover validation
12. rollback workflow
13. decommission workflow

This order builds the control rails before the risky migration operations.
