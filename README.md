# Ansible Role: samba_ad_member

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-samba_ad_member)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-samba_ad_member)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-samba_ad_member)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-samba_ad_member/dev.yml?branch=dev&event=push&label=dev)](https://github.com/jomrr/ansible-role-samba_ad_member/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-samba_ad_member/main.yml?branch=main&event=push&label=main)](https://github.com/jomrr/ansible-role-samba_ad_member/actions/workflows/main.yml?query=branch%3Amain)

Join Samba AD members with winbind, RFC2307 identity mapping, and managed file
shares.

## Purpose

Join Linux file servers to Active Directory with jomrr.samba.samba_join_member,
run Samba and winbind, and manage shares with POSIX permissions. The default ad
backend reads domain-wide RFC2307 identities.

## Scope

### Managed

- Samba and winbind packages, member configuration, domain join, enabled and
  running services.
- Stopped, disabled and masked NetBIOS name service; SMB uses TCP 445 by
  default.
- NSS passwd/group databases using files, systemd, and winbind.
- Authoritative Samba share definitions, concrete directories, access/default
  POSIX ACL entries, and persistent SELinux directory labels.

### Not Managed

- AD users, groups, RFC2307 allocation, DNS resolver configuration, host
  identity, clock synchronization, or firewall rules.
- PAM, local domain logins, SSSD, domain leave, automatic rejoin, or domain
  migrations.
- Windows Group Policy folder-redirection settings, client drive mappings,
  quotas, storage mounts, or recursive replacement of existing file permissions.

## Requirements

- Ansible Core >= 2.20; jomrr.samba >= 2.0.0, ansible.posix >= 2.0.0, and
  community.general >= 12.0.0.
- An existing AD domain, working AD DNS including SRV records, a stable
  hostname/FQDN, synchronized clocks, and connectivity to the DC. The role uses
  the system Samba Python bindings.
- A delegated join account and secret for the first join. Subsequent runs may
  omit the secret; the module checks local machine membership without contacting
  a DC.
- For ad, provisioned RFC2307 schema AND explicitly populated unique uidNumber
  values on users and gidNumber values on groups. With
  ad_unix_primary_group=true, also populate each user gidNumber with a mapped
  group GID. Provisioning RFC2307 does not assign these values.
- A filesystem supporting POSIX ACLs and extended attributes, including
  protected security.NTACL storage for the default acl_xattr module. Parent
  directories must allow intended users to traverse them.
- Apply to dedicated member systems. Existing authselect or other identity
  managers must not overwrite the role-managed passwd/group entries in
  nsswitch.conf.

## Dependencies

```yaml
collections:
  - name: ansible.posix
    version: '>=2.0.0'
  - name: community.general
    version: '>=12.0.0'
  - name: jomrr.samba
    version: '>=2.0.0'
```

## Role Variables

### `samba_ad_member_realm`

Type: `str`. Required: `true`.

AD DNS domain and Kerberos realm; immutable after joining.

### `samba_ad_member_domain`

Type: `str`. Required: `true`.

AD NetBIOS domain name; immutable after joining.

### `samba_ad_member_server`

Type: `str`. Required: `true`.

DNS hostname of an existing DC used for joining.

### `samba_ad_member_join_username`

Type: `str`. Required: `false`.

Domain account delegated permission to join computers.

Default:

```yaml
samba_ad_member_join_username: Administrator
```

### `samba_ad_member_join_password`

Type: `str`. Required: `false`.

Join account password from a secret store; required for the initial join or a
forced rejoin.

### `samba_ad_member_join_use_kerberos`

Type: `str`. Required: `false`.

Join authentication policy; desired permits NTLM fallback.

Default:

```yaml
samba_ad_member_join_use_kerberos: required
```

### `samba_ad_member_force_join`

Type: `bool`. Required: `false`.

Re-establish the machine account on every run; enable only for deliberate
repair.

Default:

```yaml
samba_ad_member_force_join: false
```

### `samba_ad_member_netbios_name`

Type: `str`. Required: `false`.

Member computer name, at most 15 characters; must match the host identity and
remain stable.

Default:

```yaml
samba_ad_member_netbios_name: '{{ ansible_facts.hostname | upper }}'
```

### `samba_ad_member_idmap_backend`

Type: `str`. Required: `false`.

Domain identity mapping backend; ad reads centrally assigned RFC2307 IDs.

Default:

```yaml
samba_ad_member_idmap_backend: ad
```

### `samba_ad_member_idmap_range`

Type: `str`. Required: `false`.

Inclusive domain UID/GID range for ad or rid; ad discards RFC2307 IDs outside
this range.

Default:

```yaml
samba_ad_member_idmap_range: 70000-99999
```

### `samba_ad_member_idmap_default_range`

Type: `str`. Required: `false`.

Disjoint writable tdb range for BUILTIN and unmapped domains with ad or rid.

Default:

```yaml
samba_ad_member_idmap_default_range: 65536-69999
```

### `samba_ad_member_idmap_autorid_range`

Type: `str`. Required: `false`.

Inclusive autorid pool replacing the separate tdb/domain ranges; only complete
rangesize blocks are usable.

Default:

```yaml
samba_ad_member_idmap_autorid_range: 65536-99999
```

### `samba_ad_member_idmap_autorid_rangesize`

Type: `int`. Required: `false`.

IDs per autorid domain range; reserve at least two whole ranges.

Default:

```yaml
samba_ad_member_idmap_autorid_rangesize: 10000
```

### `samba_ad_member_ad_unix_primary_group`

Type: `bool`. Required: `false`.

Read the primary GID from the user gidNumber instead of AD primaryGroupID.

Default:

```yaml
samba_ad_member_ad_unix_primary_group: true
```

### `samba_ad_member_ad_unix_nss_info`

Type: `bool`. Required: `false`.

Read unixHomeDirectory and loginShell from AD with the ad backend.

Default:

```yaml
samba_ad_member_ad_unix_nss_info: true
```

### `samba_ad_member_winbind_use_default_domain`

Type: `bool`. Required: `false`.

Allow unqualified domain names in NSS; false avoids local/domain name
collisions.

Default:

```yaml
samba_ad_member_winbind_use_default_domain: false
```

### `samba_ad_member_template_homedir`

Type: `str`. Required: `false`.

Home path for rid/autorid and AD accounts without unixHomeDirectory; does not
create directories.

Default:

```yaml
samba_ad_member_template_homedir: /home/%D/%U
```

### `samba_ad_member_template_shell`

Type: `str`. Required: `false`.

Login shell for rid/autorid and AD accounts without loginShell; does not
configure PAM.

Default:

```yaml
samba_ad_member_template_shell: /bin/bash
```

### `samba_ad_member_global_options`

Type: `dict`. Required: `false`.

Additional native global smb.conf options with scalar values; role-owned
identity/idmap settings take precedence.

Default:

```yaml
samba_ad_member_global_options:
  server min protocol: SMB3
  server signing: mandatory
  server smb encrypt: required
  client signing: required
  map to guest: Never
  restrict anonymous: 2
  disable netbios: true
  smb ports: '445'
  load printers: false
  disable spoolss: true
  printing: bsd
  printcap name: /dev/null
  logging: file
  log file: /var/log/samba/log.samba
  log level: 1 auth_audit:4
  max log size: 10000
  winbind refresh tickets: true
  winbind enum users: false
  winbind enum groups: false
```

### `samba_ad_member_share_options`

Type: `dict`. Required: `false`.

Native options applied to every share and overridden by each share options
dictionary; scalar values, Samba lists as strings.

Default:

```yaml
samba_ad_member_share_options:
  browseable: true
  read only: true
  guest ok: false
  inherit acls: true
  map acl inherit: true
  store dos attributes: true
  vfs objects: full_audit acl_xattr streams_xattr recycle
  acl_xattr:ignore system acls: false
  recycle:repository: .recycle
  recycle:keeptree: true
  recycle:versions: true
  recycle:directory_mode: '0770'
  recycle:subdir_mode: '0770'
  full_audit:prefix: samba_audit|%u|%I|%S
  full_audit:success: connect disconnect create_file mkdirat renameat unlinkat fset_nt_acl
  full_audit:failure: all
  full_audit:syslog: false
```

### `samba_ad_member_manage_directory`

Type: `bool`. Required: `false`.

Manage share roots by default; item manage_directory overrides this policy.
Explicit additional directories are always managed.

Default:

```yaml
samba_ad_member_manage_directory: true
```

### `samba_ad_member_directory_owner`

Type: `str`. Required: `false`.

Default owner for managed directories; accepts local names, qualified AD names,
or numeric IDs.

Default:

```yaml
samba_ad_member_directory_owner: root
```

### `samba_ad_member_directory_group`

Type: `str`. Required: `false`.

Default group for managed directories; accepts local names, qualified AD names,
or numeric IDs.

Default:

```yaml
samba_ad_member_directory_group: root
```

### `samba_ad_member_directory_mode`

Type: `str`. Required: `false`.

Default POSIX directory mode including the access ACL mask; keep it consistent
with named ACL permissions.

Default:

```yaml
samba_ad_member_directory_mode: '0770'
```

### `samba_ad_member_directory_acl_state`

Type: `str`. Required: `false`.

Default state for declared POSIX ACL entries; absent revokes the named entries.

Default:

```yaml
samba_ad_member_directory_acl_state: present
```

### `samba_ad_member_directory_acls`

Type: `list`. Required: `false`.

Default POSIX ACL entries for managed directories; item acls replaces this list.
Unlisted ACL entries are preserved.

Default:

```yaml
samba_ad_member_directory_acls: []
```

### `samba_ad_member_directory_setype`

Type: `str`. Required: `false`.

Persistent SELinux type for managed directory trees on SELinux-enabled hosts.

Default:

```yaml
samba_ad_member_directory_setype: samba_share_t
```

### `samba_ad_member_shares`

Type: `list`. Required: `false`.

Shares with root permissions and optional additional directories; removing
entries preserves stored data.

Default:

```yaml
samba_ad_member_shares: []
```

## Managed Files

- `/etc/samba/smb.conf (complete file; previous version backed up)`
- `/etc/nsswitch.conf (passwd and group entries)`
- `Share roots and additional directories declared in samba_ad_member_shares,
  with their POSIX ACL entries`
- `Persistent SELinux file-context rules for declared directories when SELinux
  is enabled`

## Check Mode

Ansible check mode predicts supported changes; the join module does not perform
a join.

- A full first-run check cannot resolve domain directory owners or validate a
  configuration before its packages and join exist. Run check mode against a
  converged member.
- force_join=true deliberately rejoins on every normal run and is not
  idempotent. Leave it false during routine management.

## Service Behavior

Configuration and join changes restart winbind and Samba. Winbind becomes
available before AD directory owners and ACL principals are resolved.

### Handlers

- restart winbind
- restart samba
- relabel shared directories

## Security Notes

- Secure defaults require SMB3 (the SMB3_11 alias), SMB encryption and server
  signing, deny guests and anonymous IPC$ access, disable printing and NetBIOS,
  and require signing in Samba client tools. Clients without SMB 3.1.1 and
  encryption support cannot connect.
- Authentication/authorization events use log level "1 auth_audit:4" and file
  logging to /var/log/samba/log.samba with a 10000 KiB rollover limit.
  full_audit also records selected file opens, namespace/ACL changes and failed
  VFS operations there. Protect and centrally collect logs; configure retention
  and storage monitoring separately.
- Settings, reasons and departures from the cited recommendations are described
  in [docs/security.md](docs/security.md), with references to Samba, BSI, CIS
  and Windows signing guidance.
- Restrict allowed hosts/networks with deployment-specific global hosts
  allow/hosts deny options and firewall policy, including IPv6 and loopback. No
  generic network allow-list can identify trusted clients.
- Replacing global_options replaces the entire default dictionary under normal
  Ansible precedence; preserve its security settings when adding native options.
  A share cannot relax globally required encryption. Mixed encryption policies
  require a deliberate global policy change.
- NTLM policy for domain users belongs on the DCs. The member ntlm auth
  parameter only governs local passdb authentication. The Kerberos join setting
  is not a Kerberos-only SMB client policy.
- Join credentials belong in Vault or another secret store; the join task is
  redacted.
- Shares default to read-only and deny guest access. Share access controls and
  filesystem ACLs both apply; write list alone cannot grant filesystem write
  permission.
- Directory management is non-recursive. Removing shares or directory entries
  preserves data. ACLs are additive unless a listed entry has state: absent;
  omitting an old ACL does not revoke it.

## Operational Notes

- The minimum server protocol defaults to SMB3 (Samba alias SMB3_11).
  defaults/main.yml contains the defaults and a compact share example; the
  argument reference and examples below describe the full model.
- Share items require name and path. owner, group, mode, acls and setype manage
  the share root; options contains native Samba settings. Each optional property
  falls back to its role-wide default. manage_directory defaults to
  samba_ad_member_manage_directory (true). Set it false for externally managed
  roots or dynamic %U/%S paths; explicit additional directories are still
  managed.
- A share directories list declares additional paths with owner, group, mode,
  acls and setype overrides. Relative paths use a concrete share path; absolute
  paths support common parents, external snapshot directories and dynamic
  shares. Each item independently uses role-wide defaults, not the share root
  overrides. Declare explicit parents before children. Roots are managed first,
  then additional directories.
- ACL entries require etype (user, group, mask or other); entity selects a named
  principal or is empty for base entries. permissions declares access, default:
  true declares inheritance, and state: absent revokes an entry without
  permissions. An item acls list replaces the role-wide list; acls: [] applies
  no entries and does not remove existing or filesystem-inherited ACLs.
- Identity mapping: ad uses uidNumber/gidNumber unchanged; idmap_range is an
  inclusive filter, not an allocator. Defaults reserve 65536-69999 for the tdb
  catch-all and 70000-99999 for the domain; RFC2307 values must fall inside the
  domain range. These lie above the usual login.defs UID_MAX/GID_MAX of 60000
  and reserved IDs 65534/65535, and below the usual subordinate-ID allocation
  starting at 100000. Custom login.defs limits, existing accounts and
  subuid/subgid assignments still need disjoint ranges. Configure ID ranges
  consistently on every member.
- rid derives IDs from RIDs and the domain range start. Identical configuration
  gives consistent IDs for that domain, but ignores RFC2307 values. autorid
  allocates ranges locally and does not guarantee identical IDs across
  independently initialized members; preserve and back up autorid.tdb. Neither
  backend migrates existing file ownership.
- autorid replaces the separate tdb/domain ranges with the pool 65536-99999 and
  rangesize=10000. Samba uses three whole blocks spanning 65536-95535; the
  remaining 4464 IDs are unused. BUILTIN and local/well-known SID mappings also
  consume blocks, and larger RIDs need extension blocks. The compact pool has
  limited capacity for additional domains or extensions. The upstream rangesize
  of 100000 cannot fit the required minimum of two blocks into this pool.
- Do not change realm, domain, netbios_name, backend, or ranges on an
  established file server without planning identity and ownership migration.
  force_join is only for explicitly repairing the existing machine trust.
- Shares use native smb.conf option names. Values are strings, numbers, or YAML
  booleans (rendered as yes/no); Samba lists are strings with native quoting,
  for example valid users: '@"EXAMPLE\File Readers" @"EXAMPLE\File Writers"'.
  Item options merge over samba_ad_member_share_options. The path field is
  authoritative. Role-owned global identity settings follow global_options and
  take precedence.
- testparm validates the candidate file before installation. It is a
  syntax/consistency check, not an access test; some unknown options only
  produce warnings. VFS-specific options require their corresponding module and
  any additional distribution packages.
- The former separate directory list is replaced by share root properties and
  nested directories. Existing shares now manage their roots by default;
  preserve their intended owner, group, mode and ACLs when migrating, or
  explicitly disable root management. For multiple exports of one directory,
  give one share responsibility for its permissions. Shared parents must allow
  traversal; define their own policy explicitly instead of relying on
  file-module parent creation.
- POSIX ACLs are the default. Set a suitable access mask through directory mode
  (the group mode bits); ACL tasks preserve that mask. For inherited default
  ACLs, explicitly declare owner, owning group, mask, and other entries
  alongside named principals to make inheritance clear. Existing children retain
  their permissions.
- The default VFS stack is full_audit acl_xattr streams_xattr recycle, with
  acl_xattr:ignore system acls: false, so POSIX permissions remain enforced. Set
  vfs objects in share_options or an individual share options dictionary to an
  ordered, space-separated module list. This replaces the entire stack; retain
  the default modules when adding others. Keep full_audit first to observe
  operations before recycle transforms deletions. An empty string disables the
  stack. Module parameters use native Samba names in the same dictionary.
- streams_xattr stores alternate data streams such as client-supplied
  Zone.Identifier in user.DosStream.* xattrs. Keep its native prefix and
  stream-type defaults. Filesystem xattr limits, including on Btrfs, constrain
  stream sizes; this supports small Windows metadata, not arbitrary NTFS stream
  sizes. Backups and local file copies must preserve xattrs. The module does not
  create zone information.
- Keep streams_xattr before recycle: the reverse order aborts smbd when deleting
  a file with ADS on the tested Samba 4.24.6 systems. With the selected order,
  Samba removes ADS before recycling the base file. Recovery from .recycle
  restores ordinary file data without its streams, including Zone.Identifier.
  This loss is deliberately accepted; see [docs/security.md](docs/security.md)
  for the rationale and sources.
- Each share uses .recycle with preserved paths and versioning. Group drives
  have a common bin; user-specific shares keep it inside the private share root.
  The 0770 creation modes allow group permissions and inherited named ACLs,
  excluding others. Samba creates the bin on first deletion; it and its nested
  paths automatically inherit the share directory default ACLs and setgid group.
  No separate recycle directory or ACL configuration is needed. Recycled files
  retain their ownership and ACLs: reading requires file access, restoring into
  the live tree requires write access.
- For private user folders under a common, non-writable share root, use
  recycle:repository: "%U/.recycle". A share already rooted in the private user
  directory uses the default .recycle without an override. The deleting user
  needs a writable destination on the same filesystem. Destination failures can
  cause permanent deletion; recycle is not a backup or retention system.
  Creation modes and default ACL inheritance affect new directories, without
  rewriting existing permissions. Previous repositories are not renamed or
  merged automatically.
- Leading-dot names are valid in Windows 11. Samba marks .recycle hidden by
  default; Explorer can open the UNC path or show hidden items. Setting hide dot
  files=false for the share makes all leading-dot entries visible.
- full_audit:syslog=false sends records through Samba logging at debug level 1.
  The defaults record connect, disconnect, create_file, mkdirat, renameat,
  unlinkat and fset_nt_acl successes and all VFS failures. Successful I/O chunks
  are not individually logged. Native operation names are Samba-version
  dependent; invalid names refuse share connections and need a functional
  connection check.
- acl_xattr forces dos filemode=true, allowing writers to change ACLs through
  SMB. Coordinate Windows ACL edits with Ansible, which reapplies declared POSIX
  entries. Samba can use POSIX-derived permissions when stored ACL hashes no
  longer match. For a deliberately Windows-only share, ignore system acls=true
  bypasses the POSIX policy. Keep the protected security.NTACL attribute name in
  either model.
- Folder redirection uses separate private user directories, e.g.
  \\fileserver\Redirected$\alice\Documents. Configure the corresponding Windows
  GPO separately. No automatic root preexec command or world-writable directory
  creation is installed.
- Use read list and write list with POSIX named-group ACLs for group drives.
  write list wins when a user belongs to both lists. Default ACLs control
  inheritance for new files and directories.

## Supported Platforms

| OS Family | Distribution | Version | Container Image |
| --------- | ------------ | ------- | --------------- |
| RedHat | AlmaLinux | latest | [jomrr/molecule-almalinux:latest](https://hub.docker.com/r/jomrr/molecule-almalinux) |
| Debian | Debian | latest | [jomrr/molecule-debian:latest](https://hub.docker.com/r/jomrr/molecule-debian) |
| RedHat | Fedora | latest | [jomrr/molecule-fedora:latest](https://hub.docker.com/r/jomrr/molecule-fedora) |
| Suse | OpenSuse Tumbleweed | latest | [jomrr/molecule-opensuse-tumbleweed:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-tumbleweed) |
| Debian | Ubuntu | latest | [jomrr/molecule-ubuntu:latest](https://hub.docker.com/r/jomrr/molecule-ubuntu) |

## Example Playbook

### Join with RFC2307 identities

Minimal membership; the host resolver already uses AD DNS.

```yaml
---
- name: Configure AD file servers
  hosts: fileservers
  gather_facts: true
  roles:
    - role: jomrr.samba_ad_member
      samba_ad_member_realm: AD.EXAMPLE.COM
      samba_ad_member_domain: EXAMPLE
      samba_ad_member_server: dc1.ad.example.com
      samba_ad_member_join_password: "{{ vault_samba_join_password }}"
```

### Group drive with readers and writers

Both AD groups have gidNumber values inside the configured ad range. ACLs are
applied after winbind is running. Samba creates `.recycle` automatically on
first deletion. The bin and its nested paths inherit the drive's default
ACLs: readers can retrieve readable deleted files; writers can recycle files
and restore them into the drive. No separate bin definition is needed.

```yaml
samba_ad_member_shares:
  - name: Projects
    path: /srv/samba/projects
    mode: '2770'
    acls:
      - {etype: group, entity: 'EXAMPLE\File Readers', permissions: r-x}
      - {etype: group, entity: 'EXAMPLE\File Writers', permissions: rwx}
      - {etype: user, permissions: rwx, default: true}
      - {etype: group, permissions: '---', default: true}
      - {etype: group, entity: 'EXAMPLE\File Readers', permissions: r-x,
         default: true}
      - {etype: group, entity: 'EXAMPLE\File Writers', permissions: rwx,
         default: true}
      - {etype: mask, permissions: rwx, default: true}
      - {etype: other, permissions: '---', default: true}
    directories:
      - path: /srv/samba
        mode: '0755'
        acls: []
    options:
      comment: Shared projects
      valid users: '@"EXAMPLE\File Readers" @"EXAMPLE\File Writers"'
      read list: '@"EXAMPLE\File Readers"'
      write list: '@"EXAMPLE\File Writers"'
      create mask: '0660'
      directory mask: '0770'
      access based share enum: true
      hide unreadable: true
```

### Private folders for Windows folder redirection

Example for alice; expand the concrete directory list for each account. Point
the GPO at Redirected$\%USERNAME%\Documents or Pictures.

```yaml
samba_ad_member_shares:
  - name: Redirected$
    path: /srv/samba/redirected
    mode: '0711'
    acls: []
    directories:
      - path: /srv/samba
        mode: '0755'
        acls: []
      - path: alice
        owner: 'EXAMPLE\alice'
        mode: '0700'
        acls: []
      - path: alice/Documents
        owner: 'EXAMPLE\alice'
        mode: '0700'
        acls: []
      - path: alice/Pictures
        owner: 'EXAMPLE\alice'
        mode: '0700'
        acls: []
    options:
      browseable: false
      read only: false
      valid users: '@"EXAMPLE\Domain Users"'
      csc policy: documents
      recycle:repository: '%U/.recycle'
      hide unreadable: true
      create mask: '0600'
      directory mask: '0700'
```

### Share rooted in the private user directory

Reuse the private directories from the previous example. Each connection
starts in its authenticated user's directory. The inherited repository
`.recycle` therefore belongs to that user without a repository override.
Disable management of the dynamic root. To manage concrete directories in
this share instead, add their absolute paths under `directories`.
For alice, `\\fileserver\Personal\Documents` resolves to
`/srv/samba/redirected/alice/Documents`; the bin is
`/srv/samba/redirected/alice/.recycle`.

```yaml
samba_ad_member_shares:
  - name: Personal
    path: /srv/samba/redirected/%U
    manage_directory: false
    options:
      browseable: false
      read only: false
      valid users: 'EXAMPLE\%U'
      create mask: '0600'
      directory mask: '0700'
```

### Btrfs snapshots for Windows Previous Versions

This layout assumes an existing Btrfs filesystem mounted at `/srv/samba`.
Create a dedicated live subvolume and keep its snapshots outside that
subvolume. Run these setup commands as root before applying the role:

```console
btrfs subvolume create /srv/samba/projects
install -d -m 0755 /srv/samba/.snapshots/projects
```

Start with the complete Projects share above, retaining its root permissions,
ACLs and existing `directories` entry. Add the two absolute snapshot paths
shown below to that share's `directories` and merge the shadow options into
its `options`. The role also labels these paths on SELinux-enabled hosts. Establish
file labels before creating read-only snapshots. After the initial
permissions and data are in place, create a read-only snapshot as root:

```sh
btrfs subvolume snapshot -r /srv/samba/projects \
  "/srv/samba/.snapshots/projects/$(date -u +@GMT-%Y.%m.%d-%H.%M.%S)"
```

For example, the live file `/srv/samba/projects/report.txt` then has an
older version at
`/srv/samba/.snapshots/projects/@GMT-2026.01.01-00.00.00/report.txt`.
`shadow:basedir` equals the snapshotted subvolume, so Samba adds no extra
`projects/` component below the timestamp. Snapshot names use UTC and
periods between time fields, avoiding Windows-invalid colons.
`shadow:fixinodes` gives snapshot files distinct inode identifiers for
Windows restore/copy operations. The snapshot parent directories must
allow traversal; the snapshots retain the original POSIX and Windows ACLs.

In Windows Explorer, open the file or folder properties and select
**Previous Versions / Vorgängerversionen**. Open or copy a historical
version, or restore it when the live destination is writable. Read-only
Btrfs snapshots prevent modification of the historical data. Historical
ACLs remain in snapshots when live ACLs change; account and share access
policy must also cover access to older data.

`shadow_copy2` exposes snapshots that already exist. Snapshot schedules,
retention and Btrfs storage belong to separate storage management.
Snapshots are not recursive across nested subvolumes and do not replace
backups. The `btrfs` VFS module is optional for clone/compression offload;
its experimental FSRVP snapshot manipulation is not enabled here.
[Snapper's](https://doc.opensuse.org/documentation/tumbleweed/snapper/)
numeric `.snapshots/<id>/snapshot` layout cannot be substituted
unchanged for this timestamp layout.

Sources: [Samba shadow_copy2](https://www.samba.org/samba/docs/current/man-html/vfs_shadow_copy2.8.html),
[Samba btrfs](https://www.samba.org/samba/docs/current/man-html/vfs_btrfs.8.html),
[Btrfs subvolumes](https://btrfs.readthedocs.io/en/latest/btrfs-subvolume.html).

```yaml
# Projects directories, including the existing parent entry:
directories:
  - path: /srv/samba
    mode: '0755'
    acls: []
  - path: /srv/samba/.snapshots
    mode: '0755'
    acls: []
  - path: /srv/samba/.snapshots/projects
    mode: '0755'
    acls: []
# Merge into the Projects share's options dictionary:
options:
  vfs objects: full_audit shadow_copy2 acl_xattr streams_xattr recycle
  shadow:snapdir: /srv/samba/.snapshots/projects
  shadow:basedir: /srv/samba/projects
  shadow:format: '@GMT-%Y.%m.%d-%H.%M.%S'
  shadow:localtime: false
  shadow:sort: desc
  shadow:fixinodes: true
```

### Alternative backends

Choose before joining and storing data. These alternatives do not read RFC2307
IDs.

```yaml
# Deterministic RID mapping with the same range on every member:
samba_ad_member_idmap_backend: rid
samba_ad_member_idmap_range: 70000-99999
samba_ad_member_idmap_default_range: 65536-69999

# Alternatively, automatic local range allocation:
# samba_ad_member_idmap_backend: autorid
# samba_ad_member_idmap_autorid_range: 65536-99999
# samba_ad_member_idmap_autorid_rangesize: 10000
```

## References

- [Samba idmap_ad](https://www.samba.org/samba/docs/current/man-html/idmap_ad.8.html)
- [Samba idmap_rid](https://www.samba.org/samba/docs/current/man-html/idmap_rid.8.html)
- [Samba idmap_autorid](https://www.samba.org/samba/docs/current/man-html/idmap_autorid.8.html)
- [Samba autorid block allocation](https://github.com/samba-team/samba/blob/master/source3/winbindd/idmap_autorid.c)
- [login.defs defaults](https://man7.org/linux/man-pages/man5/login.defs.5.html)
- [Linux and systemd UID/GID ranges](https://systemd.io/UIDS-GIDS/)
- [Samba configuration options](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html)
- [Samba acl_xattr](https://www.samba.org/samba/docs/current/man-html/vfs_acl_xattr.8.html)
- [Samba streams_xattr](https://www.samba.org/samba/docs/current/man-html/vfs_streams_xattr.8.html)
- [Samba recycle](https://www.samba.org/samba/docs/current/man-html/vfs_recycle.8.html)
- [Samba full_audit](https://www.samba.org/samba/docs/current/man-html/vfs_full_audit.8.html)
- [Samba shadow_copy2](https://www.samba.org/samba/docs/current/man-html/vfs_shadow_copy2.8.html)
- [Windows file naming](https://learn.microsoft.com/en-us/windows/win32/fileio/naming-a-file)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
