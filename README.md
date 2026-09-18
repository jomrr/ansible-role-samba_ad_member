# Ansible Role: samba_ad_member

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-samba_ad_member)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-samba_ad_member)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-samba_ad_member)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-samba_ad_member/dev.yml?branch=dev&label=dev)](https://github.com/jomrr/ansible-role-samba_ad_member/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-samba_ad_member/main.yml?branch=main&label=main)](https://github.com/jomrr/ansible-role-samba_ad_member/actions/workflows/main.yml?query=branch%3Amain)

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
- System Kerberos configuration with the AD default realm, DNS KDC discovery and
  system keytab path.
- Machine keytab creation and synchronization; automatic application of AD
  machine GPOs by winbind.
- Stopped, disabled and masked NetBIOS name service; SMB uses TCP 445 by
  default.
- NSS passwd/group databases using files, systemd, and winbind.
- Authoritative Samba share definitions, static share roots, access/default
  POSIX ACL entries, and persistent SELinux directory labels.

### Not Managed

- AD users, groups, RFC2307 allocation, DNS resolver configuration, host
  identity, clock synchronization, or firewall rules.
- PAM, local domain logins, SSSD, domain leave, automatic rejoin, or domain
  migrations.
- Windows share and filesystem ACLs, AD home/profile path assignments, and
  Windows client GPOs.
- Client drive mappings, quotas, storage mounts, or recursive replacement of
  existing file permissions.

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

### `samba_ad_member_keytab_path`

Type: `path`. Required: `false`.

System keytab to initialize and protect, also configured as default_keytab_name
in /etc/krb5.conf.

Default:

```yaml
samba_ad_member_keytab_path: /etc/krb5.keytab
```

### `samba_ad_member_global_options`

Type: `dict`. Required: `false`.

Additional native global smb.conf options with scalar values; role-owned
identity/idmap settings take precedence.

Default:

```yaml
samba_ad_member_global_options:
  kerberos method: secrets and keytab
  apply group policies: true
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

### `samba_ad_member_share_owner`

Type: `str`. Required: `false`.

Default owner for share roots; accepts local names, qualified AD names, or
numeric IDs.

Default:

```yaml
samba_ad_member_share_owner: root
```

### `samba_ad_member_share_group`

Type: `str`. Required: `false`.

Default group for share roots; accepts local names, qualified AD names, or
numeric IDs.

Default:

```yaml
samba_ad_member_share_group: root
```

### `samba_ad_member_share_mode`

Type: `str`. Required: `false`.

Default POSIX share root mode including the access ACL mask; keep it consistent
with named ACL permissions.

Default:

```yaml
samba_ad_member_share_mode: '0770'
```

### `samba_ad_member_share_acls`

Type: `list`. Required: `false`.

Default POSIX ACL entries for share roots; item acls replaces this list.
Unlisted ACL entries are preserved.

Default:

```yaml
samba_ad_member_share_acls: []
```

### `samba_ad_member_share_setype`

Type: `str`. Required: `false`.

Persistent SELinux type for share directory trees on SELinux-enabled hosts.

Default:

```yaml
samba_ad_member_share_setype: samba_share_t
```

### `samba_ad_member_shares`

Type: `list`. Required: `false`.

Shares with permissions for their static root directory; removing entries
preserves stored data.

Default:

```yaml
samba_ad_member_shares: []
```

## Managed Files

- `/etc/samba/smb.conf (complete file; previous version backed up)`
- `/etc/krb5.conf (complete file; previous version backed up), with
  /etc/krb5.conf.d snippets retained`
- `/etc/nsswitch.conf (passwd and group entries)`
- `Machine keytab at samba_ad_member_keytab_path (default /etc/krb5.keytab;
  Samba owns its contents)`
- `Static share roots declared in samba_ad_member_shares, with their POSIX ACL
  entries`
- `Persistent SELinux file-context rules for share roots when SELinux is
  enabled`

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

- See [Security considerations](docs/security.md) for security defaults,
  decisions and sources.

## Operational Notes

- The role manages each share root up to the first path component containing a
  Samba substitution: /srv/samba/profiles/%U/Documents manages
  /srv/samba/profiles. Root permission overrides apply there; shares using the
  same root must agree on those settings. The full path is preserved in
  smb.conf.
- Root permissions are non-recursive; removing a share preserves its data.
  Undeclared POSIX ACL entries are preserved: revoke entries with state: absent.
  An empty acls list applies no entries. The role takes the access ACL mask from
  the directory mode and does not recalculate it.
- Share options merge over samba_ad_member_share_options. The share path and
  role-owned global identity/idmap settings take precedence over native options.

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

Both AD groups have gidNumber values inside the configured ad range.
Access ACLs control the root; default ACLs pass reader/writer permissions
to new content, including the common recycle bin.

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

### Home folders provisioned through ADUC

The role prepares the common share root. Configure its Windows ACLs separately
following [Samba User Home Folders](https://wiki.samba.org/index.php/User_Home_Folders#Using_Windows_ACLs).
ADUC can then create the user's home folder and permissions when assigning
a path such as `\\server\users\alice`.

`Unix Admins` is an existing delegated administration group with a `gidNumber`
in the configured range. It provides initial access to configure the root ACLs.

```yaml
samba_ad_member_shares:
  - name: users
    path: /srv/samba/users
    owner: root
    group: 'EXAMPLE\Unix Admins'
    mode: '0770'
    acls: []
    options:
      read only: false
      csc policy: documents
      recycle:repository: '%U/.recycle'
```

### Roaming Windows profiles

Configure the common root's Windows ACLs separately following
[Samba Roaming Windows User Profiles](https://wiki.samba.org/index.php/Roaming_Windows_User_Profiles#Using_Windows_ACLs),
including permission to create profile folders and private inheritance.
Use the same mapped administration group as in the home-folder example.
The shares remain browsable for initial ACL administration.

Assign `\\server\profiles\%USERNAME%` through AD or a Windows GPO, without
a profile-version suffix. Windows creates the versioned profile folders.
Keep profile storage separate from redirected Documents/Pictures; caching
is disabled on this share. Recycle repositories are private to each user.

```yaml
samba_ad_member_shares:
  - name: profiles
    path: /srv/samba/profiles
    owner: root
    group: 'EXAMPLE\Unix Admins'
    mode: '0770'
    acls: []
    options:
      read only: false
      csc policy: disable
      recycle:repository: '%U/.recycle'
```

### Btrfs snapshots for Windows Previous Versions

This layout assumes an existing Btrfs filesystem mounted at `/srv/samba`.
Create a dedicated live subvolume and keep its snapshots outside that
subvolume. Run these setup commands as root before applying the role:

```console
btrfs subvolume create /srv/samba/projects
install -d -m 0755 /srv/samba/.snapshots/projects
```

Start with the complete Projects share above, retaining its root permissions
and ACLs, and merge the shadow options below into its `options`.
Storage provisioning must manage permissions and SELinux labels for the
external snapshot directories. Establish file labels before creating
read-only snapshots. After the initial
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
