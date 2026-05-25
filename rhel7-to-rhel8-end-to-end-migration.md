# RHEL 7 to RHEL 8 AWS Migration - End-to-End Specification

## 1. Purpose

This document defines the minimum required process for migrating RHEL 7 hosts to RHEL 8 in AWS using Ansible Automation Platform (AAP). It keeps only the steps and artifacts required to make migration decisions, execute cutover safely, validate success, and support rollback.

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
9. Rollback semantics must be accepted by the application owner before technical work begins.
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
- validation reports

Format:

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
phase-2-audit/
manifest/proposed/
manifest/approved/
phase-3-build/
phase-4-migrate/
validate-pre/
phase-6-cutover/
validate-post/
phase-8-rollback/
phase-9-decommission/
```

The bucket must have:

- SSE-KMS using the required KMS key
- versioning enabled
- bucket policy requiring TLS
- bucket policy requiring the expected KMS encryption

### 3.4 Approved Manifest

The approved manifest is stored at:

```text
s3://<bucket>/rhel-migration/<migration_id>/manifest/approved/approved.yml
```

IAM controls who may write to the approved prefix.

Any job that consumes the approved manifest must record:

- S3 bucket
- S3 key
- S3 version ID
- SHA-256 checksum
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
  git_commit_sha: ""
  execution_environment_digest: ""
  started_at: ""
  completed_at: ""
  result: ""
```

The git commit SHA and execution environment digest are required because cutover compares them to the final pre-cutover validation run.

## 5. Manifest Requirements

The manifest is the approved machine-readable contract for the migration.

Required top-level fields:

```yaml
manifest_version: 1
migration_id: ""
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
host_policy:
  fips_required: false
  crypto_policy: "DEFAULT" # DEFAULT or FIPS
  selinux_required_state: "enforcing"
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
  owner_rpo_accepted: false
root_sync_paths: []
data_volumes: []
users_and_groups: []
ssh: {}
repositories: []
services: []
dependencies: []
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

The audit identifies SSH material. Migration of credential material requires explicit manifest approval.

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

If host keys must be preserved to avoid client known-host failures, this is credential migration and must be explicitly allowed in the manifest.

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

Local application users and groups preserve UID/GID where required. Do not blindly recreate package-managed system users.

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
- setup artifact records key fingerprint, host fingerprint, timestamp, and job metadata

The authorized key on RHEL 7 must be restricted with options similar to:

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
| 4b | `rhel-migration-data-sync` | Manual or scheduled after setup | As needed before cutover |
| 5 | `rhel-migration-validate-pre` | Manual | Repeated |
| 6 | `rhel-migration-cutover` | Manual after cutover approval | Once |
| 7 | `rhel-migration-validate-post` | Auto from cutover, manual rerun allowed | Once or more |
| 8 | `rhel-migration-healthcheck` | Manual or scheduled | During rollback window |
| 8 | `rhel-migration-rollback` | Manual | Only if invoked |
| 9 | `rhel-migration-decommission` | Manual after rollback window | Once |

Approval gates are outside long-running workflows:

- audit reviewed and manifest approved
- build complete
- setup complete
- root sync current enough for validation
- pre-cutover validation passed and cutover approved
- rollback window closed

## 9. Phase 0 - Platform Readiness

This phase prepares the environment before host migrations begin.

Required:

1. AAP is configured and can reach RHEL 7 and RHEL 8 hosts.
2. AAP project syncs the automation repo from Bitbucket.
3. AAP project uses update-on-launch or equivalent fresh SCM sync behavior.
4. AAP execution environment includes required collections and Python modules.
5. Execution environment image digest is recorded.
6. S3 artifact bucket exists with SSE-KMS and versioning.
7. KMS key policy allows required roles to encrypt, decrypt, describe, create grants, and re-encrypt as needed.
8. IAM roles exist for audit, manifest approval, migration operation, and AAP execution.
9. Temporary migration network path is approved and reachable.

If Terraform is not used, the manual build validation must pass before setup begins.

## 10. Phase 1 - Planning

Purpose: lock down scope, rollback semantics, and network decisions.

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
16. Confirm application owner accepts rollback semantics.
17. Schedule maintenance window.
18. Define success criteria.
19. Define rollback triggers.
20. Lower DNS TTL to 60-300 seconds before cutover when DNS is the cutover strategy.
21. Record DNS TTL restoration requirement for Phase 9 when DNS TTL was lowered.

Output when snapshots are created:

- per-host planning record
- initial manifest skeleton

## 11. Phase 2 - Audit

Purpose: capture source state into S3 and produce a proposed manifest for review.

The audit playbook runs on RHEL 7. It must not change source state except for harmless temporary commands needed for discovery. It is the source of the proposed manifest, so unknown or failed captures must be recorded and reviewed before the manifest is approved.

### 11.1 Audit Rules

The audit playbook must:

1. capture machine-readable structured output
2. preserve numeric UID/GID values
3. identify missing commands or failed captures in the audit result
4. record whether each required capture is present, absent, or not applicable
5. treat unresolved unknowns as manifest review items
6. avoid changing services, files, firewall rules, users, groups, repositories, mounts, or SELinux state
7. avoid copying secrets into S3

### 11.2 Host and OS Baseline

Capture:

1. hostname, FQDN, release, kernel, uptime, boot parameters
2. package inventory from `rpm -qa`
3. yum repository files, yum config, enabled and disabled repos, installed package to repo mapping, yum plugins, RHSM state
4. GPG public key fingerprints from RPM DB and `/etc/pki/rpm-gpg/`
5. RHSM identity and enabled content sources when subscription-manager is used
6. enabled kernel modules, blacklist files, boot-time module options, and loaded modules relevant to the application
7. time configuration from chrony, ntp, timezone, and clock source
8. MTA configuration and relay dependencies when mail is used
9. RHEL 7 to RHEL 8 compatibility checks: Python shebangs, ntp service, MTA, authconfig/SSSD, FIPS state, database versions, Btrfs presence

### 11.3 Users, Groups, and Service Accounts

Capture all local users, groups, and service accounts, including non-login accounts.

Required user and group captures:

1. `/etc/passwd` account map: name, UID, primary GID, GECOS, home, shell
2. `getent passwd` account map for NSS-visible users referenced by services, files, cron, or processes
3. `/etc/group` group map: name, GID, members
4. `getent group` group map for NSS-visible groups referenced by services, files, cron, or processes
5. account source when known: local, SSSD, LDAP, AD, or other NSS provider
6. account lock, expiry, and password-age status without password hashes
7. home directory path, owner, group, mode, existence, and size
8. supplemental group membership
9. sudo grants from `/etc/sudoers` and `/etc/sudoers.d/`
10. accounts with interactive shells
11. accounts with no-login shells that own application files, run services, own cron jobs, or bind ports
12. UID/GID reuse, duplicates, and collisions
13. accounts outside the expected application or operating-system account ranges
14. local accounts that should not be recreated because they are package-managed on RHEL 8

Required service-account discovery:

1. systemd `User=`, `Group=`, `SupplementaryGroups=`, and `DynamicUser=` references
2. SysV init scripts and xinetd service users when present
3. running process owners grouped by unit, command, PID, and listening port
4. file owners for application paths, data paths, log paths, custom unit files, cron files, and certificate directories
5. cron, systemd timer, and at-job owners
6. sudo rules that grant service accounts elevated access
7. service-account home directory profile files such as `.bashrc`, `.bash_profile`, `.profile`, and shell-specific startup files
8. service-account `.ssh` directory owner, mode, authorized-key fingerprints, authorized-key options, SSH client config, and known-host fingerprints
9. service-account environment files referenced by systemd units
10. service-account dependencies such as local sockets, PID files, runtime directories, and writable log directories
11. service accounts that are external identities and must not be created locally unless explicitly approved in the manifest

### 11.4 Services, Schedules, and Runtime

Capture:

1. enabled, running, disabled, masked, and failed systemd units
2. custom unit files in `/etc/systemd/system/`
3. unit drop-ins from `/etc/systemd/system/*.d/`
4. unit dependencies, ordering, wanted-by targets, restart policy, working directory, environment files, PID files, and runtime directories
5. unit `ExecStart`, `ExecStop`, `ExecReload`, and pre/post command paths
6. SysV init scripts still in use
7. xinetd services when present
8. cron jobs from `/etc/cron*`, `/var/spool/cron/`, and user crontabs
9. systemd timers and their paired service units
10. at jobs
11. process table grouped by service, user, command, and parent process
12. listening ports mapped to process, user, service, protocol, and bind address
13. application inventory: versions, config paths, binary paths, data paths, log paths, runtime paths, and health-check commands
14. management, logging, monitoring, and security agents with config paths and identity-file metadata

### 11.5 Filesystem, Application Paths, and File Metadata

Capture:

1. filesystem layout, mounts, LVM topology, fstab, disk usage, UUIDs, labels, and filesystem types
2. EBS-backed data-volume candidates and root-volume paths that must be copied
3. bind mounts, network mounts, autofs maps, tmpfs mounts, and mount options
4. candidate root-sync paths under locations such as `/opt`, `/srv`, `/usr/local`, application-owned `/var` paths, custom web roots, and custom service directories
5. application config, binary, data, log, runtime, and certificate paths discovered from services and processes
6. owner, group, mode, numeric UID/GID, ACLs, xattrs, SELinux context, file capabilities, and immutable attributes for candidate paths
7. file counts, total size, largest files, sparse files, hardlinks, symlinks, device files, sockets, and FIFOs in candidate paths
8. setuid and setgid files in candidate paths
9. checksum spot-check candidates for critical configuration files
10. explicit exclude candidates such as cache, tmp, socket, PID, and generated runtime directories
11. package ownership for candidate files from `rpm -qf`
12. modified package-managed config candidates from `rpm -V` where safe and targeted
13. custom or unowned files under `/etc` that are referenced by services, processes, cron, or application configuration

Capture metadata for these important configuration areas when present:

- `/etc/fstab`
- `/etc/hosts`, `/etc/resolv.conf`, `/etc/nsswitch.conf`
- `/etc/sysconfig/`
- `/etc/default/`
- `/etc/systemd/system/`
- `/etc/profile`, `/etc/profile.d/`, and shell environment files referenced by services
- `/etc/tmpfiles.d/`
- `/etc/logrotate.d/`
- `/etc/cron*` and `/var/spool/cron/`
- `/etc/sudoers` and `/etc/sudoers.d/`
- `/etc/security/`
- `/etc/pam.d/`
- `/etc/ssh/sshd_config` and SSH daemon config fragments
- `/etc/ssh/ssh_config` and SSH client config fragments
- `/etc/sssd/`, `/etc/krb5.conf`, and `/etc/realmd.conf`
- `/etc/yum.conf`, `/etc/yum.repos.d/`, and yum plugin configuration
- `/etc/pki/` public certificate and trust-anchor metadata
- application-specific configuration files referenced by services, processes, cron, or documentation

### 11.6 Security Settings

Capture:

1. SELinux mode, configured mode, policy type, custom modules, booleans, fcontext rules, and contexts on candidate root-sync paths
2. recent SELinux AVC denials related to application services or candidate paths
3. FIPS state from system configuration and runtime checks
4. crypto policy and OpenSSL crypto configuration relevant to application clients
5. SSH daemon configuration, daemon config fragments, public host key fingerprints, and allowed authentication methods
6. PAM, authconfig/authselect, SSSD, NSS, Kerberos, LDAP, and local authentication configuration
7. password policy, faillock, pwquality, login.defs, default umask, and `/etc/security/limits*`
8. sudoers and privilege escalation paths
9. sysctl settings and custom files in `/etc/sysctl.d/`
10. kernel module policy from `/etc/modprobe.d/`
11. firewalld zones, services, rich rules, direct rules, runtime state, and permanent state
12. iptables, ip6tables, and nftables rules when present
13. listening ports and inbound/outbound dependency requirements
14. auditd rules and daemon state when auditd is installed
15. local certificate trust anchors and public certificate fingerprints
16. private key, keytab, token, and secret file locations as metadata only
17. security, monitoring, and endpoint-agent service state, package version, config path, and identity-file metadata

### 11.7 SSH Key Material and Trust

Capture SSH key material safely. Do not copy private keys or full public key bodies into sanitized artifacts.

Required SSH captures:

1. effective `sshd_config`, all included daemon config fragments, and the resolved values of `AuthorizedKeysFile`, `AuthorizedKeysCommand`, `TrustedUserCAKeys`, `RevokedKeys`, `HostKey`, `HostCertificate`, `HostbasedAuthentication`, `PubkeyAuthentication`, `PasswordAuthentication`, `PermitRootLogin`, `AllowUsers`, `AllowGroups`, `DenyUsers`, and `DenyGroups`
2. SSH host public key fingerprints for each configured host key algorithm
3. SSH host private key file metadata only: path, owner, group, mode, size, modified time, algorithm when derivable from matching public key, and whether a matching public key exists
4. SSH host certificates, host CA references, key revocation lists, and principal files as metadata plus public fingerprints where safe
5. root and service-account `.ssh` directory owner, group, mode, and SELinux context
6. authorized-key files discovered from `AuthorizedKeysFile`, not only the default `.ssh/authorized_keys`
7. authorized-key fingerprints, key type, comment, options, source file, owning account, and file permissions
8. authorized-key command restrictions, source restrictions, expiry options, principals, and forced commands
9. duplicate authorized keys across accounts
10. authorized keys for disabled, locked, expired, or no-login accounts that still own services or files
11. service-account SSH client config from `~/.ssh/config`, `/etc/ssh/ssh_config`, and included client config fragments
12. known-host files from service-account homes, root home, and system known-host files
13. known-host entry metadata: key type, fingerprint, hashed-host indicator, marker such as `@cert-authority` or `@revoked`, source file, owner, group, and mode
14. SSH private key file locations for root and service accounts as metadata only, including path, owner, group, mode, size, modified time, encrypted-or-unknown status, and matching public-key fingerprint when safe to derive without exporting the private key
15. SSH agent or socket references in service environment files, unit files, cron, or shell profile files
16. SSH-related application dependencies, such as scp, sftp, rsync-over-ssh, Git over SSH, database tunnels, or batch jobs
17. host-key preservation requirement candidates, including clients or known-host entries that would break if host keys change
18. SSH findings that require manifest review: private keys present, weak key types, loose file permissions, missing public keys, unknown key owner, locked account with authorized keys, or host-key preservation requested

Sanitized SSH output records fingerprints and metadata only. Full authorized-key bodies, private keys, keytabs, passphrases, and credentials are not copied.

### 11.8 Network and Dependency Captures

Capture:

1. interface names, MAC addresses, private IPs, secondary IPs, routes, DNS servers, search domains, and resolver behavior
2. NetworkManager profiles, legacy network-scripts, and interface-specific routes
3. hostname and reverse-DNS expectations
4. upstream and downstream application dependencies, ports, protocols, and owners
5. load balancer health-check path, port, and expected response when load balancer cutover is used
6. Route 53 records, EIP allocation, or secondary private IP details for the selected cutover strategy
7. outbound firewall or proxy requirements
8. local hosts-file dependencies
9. application allowlists that reference source IP, hostname, certificate subject, or SSH host key

### 11.9 AWS Captures

Capture:

1. source instance ID, AZ, instance type, AMI, launch time
2. IAM instance profile, role ARN, and attached policy names
3. security groups and rules
4. subnet route table and NACLs that affect RHEL 8 to RHEL 7 migration traffic
5. full tag set
6. ENI IDs, subnet, primary IP, secondary IPs, EIP association, source/destination check, and attachment details
7. attached EBS volumes with size, type, IOPS, throughput, encryption, KMS key, attachment name, delete-on-termination flag
8. LVM mapping to EBS volumes
9. relevant agent registration state when agents must be re-identified on RHEL 8
10. relevant Route 53 records, EIP allocations, or load balancer target group memberships depending on cutover strategy
11. KMS keys required for the root volume, data volumes, and any application volume operations

### 11.10 Proposed Manifest Derivation

The audit playbook must generate a proposed manifest containing:

1. source identity, AWS placement, network identity, IAM profile, and security groups
2. target placement requirements, including same-AZ requirement
3. proposed root-sync paths with owner service, reason, size, attributes, and exclude candidates
4. data-volume records with AWS identity, Linux identity, filesystem, mount point, LVM details, and cutover handling
5. users and groups that must be recreated, including service accounts and exact UID/GID requirements
6. SSH migration decisions that need review, including authorized-key fingerprints, private-key metadata, known-host dependencies, host-key strategy, SSH CA references, and service-account SSH client requirements
7. repository and GPG fingerprint candidates for approval
8. services, ports, dependencies, service-account mappings, and startup order
9. security settings needed on RHEL 8
10. cutover strategy inputs
11. rollback RPO statement and rollback blockers
12. unresolved conflicts, unknowns, and hard blockers

### 11.11 Sanitization Rules

Do not copy these values into sanitized artifacts:

- SSH private keys
- SSH host private keys
- certificate private keys
- keytabs
- password hashes from `/etc/shadow`
- application passwords
- API tokens
- database connection secrets
- repository credentials
- cloud credentials

For sensitive files, record only:

- path
- owner
- group
- mode
- SELinux context
- size
- modified time
- secret type
- public fingerprint or checksum only when safe

Authorized keys and known-host entries are recorded as fingerprints, comments, markers, options, account mapping, and file metadata, not full key bodies.

### 11.12 Audit Outputs

Write to S3:

```text
phase-2-audit/sanitized/<timestamp>-sanitized.yml
manifest/proposed/<timestamp>-proposed.yml
```

The sanitized audit output must include capture status, command failures, unknowns, proposed manifest inputs, and hard blockers.

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
7. Set SELinux to the manifest-required state.
8. Apply the manifest crypto policy.
9. Configure repositories from the approved allowlist.
10. Register with subscription-manager when required for the selected repositories.
11. Confirm `dnf repolist enabled` matches the approved manifest.
12. Enable required module streams.
13. Install approved package set.
14. Create approved users and groups with exact UID/GID only after collision checks.
15. Configure sudoers, SSH daemon policy, and approved authorized keys.
16. Generate new SSH host keys unless preservation is explicitly allowed in the manifest.
17. Configure network, firewall, sysctl, time sync, auth, certificates, and agents required by the manifest.
18. Confirm KMS access required for encrypted root and data volumes.
19. Confirm AAP inventory contains the target host.
20. Run build smoke tests.

### 12.3 Build Validation

The `manual-build-validate.yml` or build validation playbook verifies:

- target is RHEL 8 expected minor version
- target is in same AZ as source
- target has expected instance type
- root volume is encrypted with approved KMS key
- IAM profile is correct
- security groups are correct
- SELinux matches manifest-required state
- FIPS and crypto policy match manifest
- approved repos are enabled
- unauthorized repos are disabled
- required packages and agents are installed
- users and groups match approved manifest
- SSH daemon policy matches approved manifest
- AAP can connect
- RHEL 8 can reach RHEL 7 on the approved rsync path
- S3 artifact upload works

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
10. upload setup artifact to S3

Outputs:

```text
phase-4-migrate/<timestamp>-setup.yml
```

Approval gate: setup complete, root sync may begin.

### 13.2 Root Sync

The data-sync workflow runs as needed during the parallel phase and must run during pre-cutover validation.

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
4. run `getcap -r` comparison for file capabilities listed in the manifest
5. upload sync report to S3

Outputs:

```text
phase-4-migrate/rsync/<timestamp>-rsync-report.yml
```

### 13.3 Data Volume Snapshots

During the parallel phase:

1. create snapshots of each data volume when the rollback plan requires them before cutover
2. tag snapshots with `migration_id`
3. record snapshot IDs and volume metadata

These snapshots are crash-consistent unless application quiescing is performed.

Outputs:

```text
phase-4-migrate/snapshots/<timestamp>-snapshot-report.yml
```

## 14. Phase 5 - Pre-Cutover Validation

Purpose: prove the RHEL 8 host is ready before production identity or data volumes move.

This workflow is run as needed during the parallel phase. The final passing run must be within the approved freshness window, default four hours.

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
13. validate required agent check-in under staging identity when agents are required
14. verify TLS and SSH client compatibility with selected crypto policy
15. verify AWS permissions expected from the instance
16. verify KMS access
17. verify cutover-strategy-specific readiness
18. verify data-volume mapping using `/dev/disk/by-id`, not `/dev/sdX`
19. verify EBS volume metadata matches manifest
20. compile validation report

Do not use broad SELinux permissive mode as the default.

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

Approval gate: cutover go/no-go.

## 15. Phase 6 - Cutover

Purpose: move production service to RHEL 8 in the maintenance window.

The cutover workflow is launched manually after cutover approval. It does not rely on a long-running parent workflow.

### 15.1 Precondition Check

The first workflow node must verify:

1. recent passing pre-cutover validation artifact exists in S3
2. validation artifact is within freshness window
3. cutover git commit SHA equals validation git commit SHA unless the cutover approval explicitly allows a newer commit
4. cutover execution environment digest equals validation digest unless the cutover approval explicitly allows a newer digest
5. approved manifest S3 version ID equals validation manifest version ID
6. approved manifest checksum equals validation manifest checksum
7. scheduled rsync, if used, is disabled or paused
8. maintenance window is active

Fail if any check fails.

### 15.2 Cutover Sequence

1. Enter maintenance window and pause monitoring alerts if configured.
2. Stop application services on RHEL 7 in dependency order.
3. Run final root-content rsync delta.
4. Run application-aware final dumps, queue drains, or exports required by the manifest.
5. Apply final dumps to RHEL 8 if produced.
6. Confirm no final rsync errors.
7. Unmount data volumes cleanly on RHEL 7.
8. Confirm no busy files remain.
9. Flush filesystem state.
10. Take final snapshot of each data volume.
11. Tag final snapshots with `migration_id` and `cutover-final=true`.
12. Stop RHEL 7 and preserve it for the rollback window.
13. Detach data volumes from RHEL 7.
14. Attach data volumes to RHEL 8.
15. On RHEL 8, discover devices by `/dev/disk/by-id`.
16. If LVM is used, run `vgscan` and `vgchange -ay`.
17. Use `vgimport` only if the volume group was explicitly exported.
18. Use `vgimportclone` only for cloned or snapshot-restored VGs with duplicate UUIDs.
19. Mount data volumes by UUID according to approved fstab.
20. Verify mounts, filesystems, sizes, and SELinux contexts.
21. Set RHEL 8 hostname to production hostname.
22. Update agent identities where required.
23. Start application services on RHEL 8 in dependency order.
24. Run immediate smoke checks.
25. Execute production identity transfer based on `cutover.strategy`.
26. Coordinate firewall or security group updates that reference the old IP.
27. Re-enable monitoring if it was paused.
28. Launch post-cutover validation.

### 15.3 Production Identity Transfer

DNS:

1. confirm the manifest contains hosted zone ID, record name, record type, source value, target value, cutover TTL, and restore TTL
2. update A or AAAA record from the RHEL 7 private IP to the RHEL 8 private IP
3. verify the authoritative Route 53 value
4. verify resolution from representative downstream consumers
5. record old value, new value, TTL, hosted zone ID, change ID, and timestamps in the cutover log
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
4. coordinate neighbor cache or ARP refresh when required by the network design

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
4. compare error rate and latency against baseline if a baseline exists
5. confirm required agents check in under production identity
6. run application-specific success checks
7. verify mounted data volumes
8. run `xfs_info` or filesystem-specific equivalent
9. verify `df` sizes match expected values
10. check SELinux denials
11. collect service status and logs
12. confirm success criteria

Outputs:

```text
validate-post/<timestamp>-report.yml
```

## 17. Phase 8 - Stabilization and Rollback

Purpose: keep rollback viable while the new host proves stable.

Stabilization steps:

1. keep RHEL 7 stopped, not terminated
2. preserve final cutover snapshots
3. run health checks during the rollback window
4. track anomalies
5. decide whether to close or extend rollback window

### 17.1 Rollback Rules

Rollback is not automatic. It requires an explicit rollback decision according to the rollback triggers.

Before rollback:

1. stop or quiesce RHEL 8 services
2. snapshot RHEL 8 root volume and attached data volumes if current-state preservation is required
3. record the exact reason for rollback
4. confirm application owner accepts the approved RPO impact

### 17.2 Rollback Sequence

1. Put monitoring into maintenance mode if configured.
2. Stop RHEL 8 application services.
3. Snapshot RHEL 8 root and data volumes if current-state preservation is required.
4. Stop RHEL 8 when required before detaching volumes.
5. Detach data volumes from RHEL 8.
6. Restore final cutover snapshots to fresh volumes.
7. Attach restored volumes to RHEL 7.
8. Mount volumes on RHEL 7.
9. Start RHEL 7.
10. Start services on RHEL 7.
11. Restore production identity based on original cutover strategy.
12. Validate service health.
13. Upload rollback report.

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
2. Tear down ephemeral rsync SSH channel.
3. Confirm migration key removed from RHEL 7 `authorized_keys`.
4. Confirm migration keypair deleted from RHEL 8.
5. Disable and delete AAP schedules created for the migration.
6. Remove temporary security group rules.
7. Remove temporary KMS grants if created.
8. Delete throwaway validation EBS volumes if created.
9. Delete or retain snapshots according to rollback policy.
10. Deregister RHEL 7 agents from consoles.
11. Remove RHEL 7 from AAP inventory.
12. Remove RHEL 7 from monitoring.
13. Terminate RHEL 7 through the approved provisioning method or manual API.
14. Release orphaned ENIs, EBS volumes, temporary IPs, and DNS records.
15. Restore DNS TTL if it was lowered.

Outputs:

```text
phase-9-decommission/<timestamp>-decommission-report.yml
```


## 19. Minimum Playbook Layout

Minimum structure:

```text
playbooks/
  audit/
    discover-migration-manifest.yml
    tasks/
      host-os-baseline.yml
      users-groups-service-accounts.yml
      services-schedules-runtime.yml
      filesystem-application-paths.yml
      security-settings.yml
      ssh-key-material.yml
      network-dependencies.yml
      aws-captures.yml
      proposed-manifest.yml
      sanitize-output.yml
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
    volume-health.yml
  validate-shared/
    agent-checkin.yml
    outbound-connectivity.yml
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

## 20. Required Operational Artifacts

Only artifacts required for migration decisions, resume, validation, or rollback are mandatory. Separate audit evidence is not required by this migration process.

| Artifact | Why it is required | S3 location |
|---|---|---|
| Approved manifest | Defines the approved migration contract consumed by automation | `manifest/approved/` |
| Manifest checksum and version | Confirms automation is using the approved manifest | `manifest/approved/` |
| Source audit output | Provides the source baseline used to create and review the manifest | `phase-2-audit/` |
| Build validation result | Confirms the RHEL 8 target is ready before migration setup | `phase-3-build/` |
| Migration setup result | Confirms the SSH channel, host fingerprint, and rsync prerequisites are ready | `phase-4-migrate/` |
| Root sync report | Confirms approved root-volume paths copied successfully | `phase-4-migrate/rsync/` |
| Snapshot report | Records snapshot IDs needed for cutover safety or rollback | `phase-4-migrate/snapshots/` when used, `phase-6-cutover/` |
| Pre-cutover validation result | Required gate before cutover | `validate-pre/` |
| Cutover log | Records production identity transfer, volume movement, and cutover status | `phase-6-cutover/` |
| Post-cutover validation result | Confirms the RHEL 8 host is serving correctly after cutover | `validate-post/` |
| Rollback report | Required only if rollback is invoked | `phase-8-rollback/` |
| Decommission report | Required to close the migration after the rollback window | `phase-9-decommission/` |

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
13. Cutover job commit SHA or execution environment digest does not match validation and the cutover approval does not explicitly allow the difference.
14. Data-volume final snapshot fails.
15. Application owner has not accepted rollback RPO semantics.

## 22. Implementation Order

Minimum build order:

1. S3/KMS/IAM artifact foundation
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
