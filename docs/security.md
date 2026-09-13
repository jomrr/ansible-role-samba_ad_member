# Security considerations

These defaults target an AD member with winbind, private redirected folders and
group shares. The table records the choices and their reasons.

| Setting or measure | Decision and reason | Sources |
| --- | --- | --- |
| Minimum dialect | `SMB3` (alias SMB3_11); require modern clients. | [Samba] |
| Maximum dialect | Leave negotiated; avoid an unnecessary upper limit. | [Samba] |
| SMB encryption | Required globally, including private and group data. | [BSI] A15; [CIS 3] |
| Server/client signing | Require signatures on Samba connections. | [Samba]; [Windows] |
| Guest and anonymous IPC access | Deny; shares require authentication. | [Samba] |
| Authentication | ADS membership with winbind. | [Samba] |
| Printing and NetBIOS | Disable unused services; SMB uses TCP 445. | [CIS 4] |
| Network access | Configure trusted networks and firewalls per site. | [BSI] A2 |
| Connection logs | `1 auth_audit:4`; bounded file rollover. | [Samba]; [CIS 8] |
| File permissions | Read-only shares by default; explicit POSIX ACLs. | [BSI] A3; [CIS 3] |
| Windows ACL storage | Enable `acl_xattr`; retain POSIX enforcement. | [ACL module] |
| Deleted files | One `.recycle` per share; access follows its users/groups. | [recycle] |
| File operation logs | Enable `full_audit` with explicit operations. | [Full audit] |
| Additional VFS modules | Explicit ordered stack and native options. | [BSI] A1/A10 |
| Identity mapping | RFC2307 `ad`; domain 70000-99999, catch-all 65536-69999. | [BSI] A6; [ID mapping]; [Login defaults] |
| Share administration | Generate and validate the complete smb.conf. | [BSI] A2/A5 |
| PAM integration | Native tools are appropriate for OS authentication. | [Authselect]; [BSI] A6 |
| DNS and time | Use existing AD infrastructure. | [BSI] A7-A9 |
| Backups | Preserve data, ACLs, xattrs, secrets and idmap state. | [BSI] A13 |

## Why some recommendations differ

[BSI] APP.3.4 A3 recommends retaining signing defaults unless local policy
differs. This role deliberately requires signing. A15 recommends SMB3 encryption
for higher protection needs; it is enabled by default here because the intended
shares contain private and group data. Windows CIS/STIG signing settings in the
[Microsoft mapping][Windows] support the same protection goal; their Windows
policy identifiers are not Linux configuration requirements. [CIS 3], [CIS 4]
and [CIS 8] provide general guidance for access, services and logging.

BSI A5 recommends registry-managed shares. This role instead owns smb.conf
through reviewed Ansible inventory and validates candidates with `testparm`.
Registry administration would introduce a second configuration authority.

## ACLs and VFS modules

`acl_xattr:ignore system acls: false` keeps POSIX permission checks active while
Windows descriptors use the protected `security.NTACL` attribute. Changing that
attribute to `user.NTACL` would allow local tampering. The module forces
`inherit acls`, `dos filemode` and `force unknown acl user` on. In particular,
writers can change permissions through SMB. [ACL module]

Ansible reapplies the declared POSIX entries on subsequent runs. Samba checks
hashes of its stored descriptors against filesystem permissions and can fall
back to a POSIX-derived descriptor after changes. This is not a recursive
rewrite of child ACLs. Coordinate Windows and Ansible administration of the same
paths. [ACL implementation]

Windows-only administration can explicitly select `ignore system acls: true`;
then SMB no longer enforces the Ansible POSIX policy. This is an alternative
access model, not a hardening switch. [ACL module]

The default stack is `full_audit acl_xattr recycle`. Configure plugins through
`vfs objects` in `samba_ad_member_share_options` or a share's `options`; each
value replaces the complete stack. Keep `full_audit` first so client operations
are observed before `recycle` handles deletions. Additional modules may require
distribution packages.

Recycle bins use `.recycle` relative to the share root, with versioning and
`0770` directory creation modes. This permits group access and inherited named
ACLs while excluding others. Samba creates the common group-drive bin on first
deletion; it and its nested paths inherit the drive's POSIX default ACLs and
setgid group ownership. No separate bin configuration is needed. Private shares
keep the bin inside the user's private directory. Redirected folders below a
common non-writable share root
use `%U/.recycle`. Recycled files retain their existing ownership and ACLs;
recycling does not make restricted files readable by every share user.
Reading a recycled file requires read access, restoring it into the live tree
requires write access. [recycle]; [Recycle implementation]

The destination must be writable by the deleting user on the same filesystem.
Creation modes and default ACL inheritance apply to new directories; they do not
repair existing directories. Creation or move failures can result in permanent
deletion. Previous repositories are not renamed or merged automatically.
Retention, space management and backups remain
separate responsibilities. [recycle]; [Recycle implementation]

`full_audit` logs selected successful opens, namespace and ACL changes, plus
failed VFS operations. `full_audit:syslog: false` routes records through Samba
logging at debug level 1. It does not record every successful I/O chunk; names
and paths in the records require appropriate access and retention controls.
Unknown operation names prevent share connections, so module changes require
functional verification as well as `testparm`. [Full audit]

The README's Btrfs example uses server-created read-only snapshots with
`shadow_copy2` for Windows Previous Versions. Historical permissions remain in
snapshots; current access policy and snapshot retention must account for this.
[Shadow copies]; [Btrfs]

## Deployment choices

The default ID ranges sit above the usual `login.defs` UID/GID maximum of 60000
and reserved IDs 65534/65535, and below the usual subordinate UID/GID allocation
starting at 100000. This avoids those default allocations; custom limits,
existing accounts and subordinate-ID assignments must still remain disjoint.
The `ad` backend filters existing RFC2307 IDs; it does not renumber accounts
outside the selected domain range. [Login defaults]; [Linux IDs]; [ID mapping]

The alternative `autorid` backend uses 65536-99999 with blocks of 10000 IDs.
Samba uses only complete blocks: three cover 65536-95535, leaving 4464 IDs unused.
BUILTIN and local/well-known SID mappings also consume blocks; additional
domains and RID extensions are limited by the remaining capacity. The upstream
block size of 100000 is too large for this pool. [Autorid]; [Autorid implementation]

Configure host allow/deny rules and the firewall from actual trusted IPv4/IPv6
networks. The role cannot infer a safe subnet. Protect and centrally collect
logs: the default file rotates to `.old` at 10000 KiB, which does not establish
retention or cover every file operation. Configure storage encryption, backups,
restoration tests and log retention separately. [BSI] A2/A13; [CIS 3]; [CIS 8]

Replacing `samba_ad_member_global_options` replaces its complete default
dictionary under normal Ansible precedence. Retain the desired protections
when adding options. Globally required encryption cannot be relaxed by a
share. Domain-user NTLM policy belongs on the DCs; the member's `ntlm auth`
setting controls local passdb authentication. [Samba]

Native PAM management uses `authselect` on Red Hat systems, `pam-auth-update`
on Debian/Ubuntu and `pam-config` on openSUSE. It avoids editing generated
authentication files directly. [Authselect]; [Debian PAM]; [SUSE PAM]

In particular, `authselect select winbind` manages both NSS and PAM and enables
domain authentication for PAM consumers; it cannot merely add winbind to NSS.
PAM logins and automatic home creation are separate choices from SMB access.
The current role configures NSS only and does not activate PAM domain logins.
Existing identity managers must not overwrite those NSS entries. [Authselect]

[Samba]: https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html
[ACL module]: https://www.samba.org/samba/docs/current/man-html/vfs_acl_xattr.8.html
[ACL implementation]: https://github.com/samba-team/samba/blob/master/source3/modules/vfs_acl_common.c
[ID mapping]: https://www.samba.org/samba/docs/current/man-html/idmap_ad.8.html
[Linux IDs]: https://systemd.io/UIDS-GIDS/
[Login defaults]: https://man7.org/linux/man-pages/man5/login.defs.5.html
[Autorid]: https://www.samba.org/samba/docs/current/man-html/idmap_autorid.8.html
[Autorid implementation]: https://github.com/samba-team/samba/blob/master/source3/winbindd/idmap_autorid.c
[recycle]: https://www.samba.org/samba/docs/current/man-html/vfs_recycle.8.html
[Recycle implementation]: https://github.com/samba-team/samba/blob/master/source3/modules/vfs_recycle.c
[Full audit]: https://www.samba.org/samba/docs/current/man-html/vfs_full_audit.8.html
[Shadow copies]: https://www.samba.org/samba/docs/current/man-html/vfs_shadow_copy2.8.html
[Btrfs]: https://btrfs.readthedocs.io/en/latest/btrfs-subvolume.html
[BSI]: https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_3_4_Samba_Edition_2023.pdf?__blob=publicationFile&v=3
[CIS 3]: https://cas8.docs.cisecurity.org/en/latest/source/Controls3/
[CIS 4]: https://cas8.docs.cisecurity.org/en/latest/source/Controls4/
[CIS 8]: https://cas8.docs.cisecurity.org/en/latest/source/Controls8/
[Windows]: https://learn.microsoft.com/en-us/azure/governance/policy/samples/guest-configuration-baseline-windows
[Authselect]: https://github.com/authselect/authselect/blob/master/src/man/authselect.8.adoc
[Debian PAM]: https://manpages.debian.org/testing/libpam-runtime/pam-auth-update.8.en.html
[SUSE PAM]: https://manpages.opensuse.org/Tumbleweed/pam-config/pam-config.8.en.html
