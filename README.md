# encryptionconsulting.certsecure

The `encryptionconsulting.certsecure` Ansible collection served by CertSecure as a
Remote Archive source (`GET /aapIntegration/playbooks/archive`) to the "CertSecure"
Project it provisions on every newly-added AAP connection (see
`docs/design/aap-integration.md`). Editing a playbook (via the API below) takes
effect the next time AAP syncs the Project -- no CertSecure backend change or
redeploy required.

## Layout and the three-tier resolution model

```
roles/                   <- disk-level override tier. EMPTY by default.
roles_reference/         <- the shipped baseline. One copy of each playbook,
                             replaced wholesale by every CertSecure release.
```

`build_playbooks_archive()` (`classes/functions/automation_platform.py`)
builds the archive AAP fetches by resolving each playbook file individually,
in this order:

1. **`roles/<path>`** on disk, if present -- a manual/emergency override.
   Not preserved across a CertSecure upgrade (it ships inside the same build
   artifact as everything else under `static/`).
2. **`roles_reference/<path>`** -- the shipped baseline.

This means there is exactly **one** file to update per playbook per
CertSecure release (`roles_reference/`), and a user's edits (in Mongo)
automatically carry forward on top of the new release without anyone having
to manually re-apply or merge them. See `roles/README.md` for the same
explanation from the disk side.

`roles_reference/`'s own file set is also what defines *which* playbook paths
exist at all (`list_playbook_paths()`) -- `roles/` and Mongo can only
override an existing path's *content*, except that a file added directly
under `roles/` with no `roles_reference/` counterpart is also picked up (see
"Adding a new custom playbook" below).

**Important: the on-disk `roles/` / `roles_reference/` directory names are a
CertSecure-internal authoring detail only -- AAP never sees them.** Inside
the archive AAP actually fetches, every file is shipped under a top-level
`playbooks/` directory instead (e.g. `playbooks/nginx_deploy/main.yml`), and
that's the prefix to use in a job template's `"playbook"` field
(`POST .../jobTemplates/create`). This was confirmed live against a real AAP
4.8 instance: AAP's own project playbook-discovery
(`GET /projects/{id}/playbooks/`, which job template creation validates the
`playbook` field against) silently excludes anything under a directory
literally named `roles` as Ansible role-internal content -- even though the
archive unpacks it there without error and the file is a perfectly valid
standalone playbook (it starts with `- hosts: all`, not a role in the strict
tasks/handlers/defaults sense). `playbooks/` isn't a name Ansible/AAP
reserves, so content there is discovered normally.

### Variable contracts (`argument_spec.yml`)

Alongside each ACME playbook's `main.yml`, and at `_acme_lib/vars/argument_spec.yml`
for the variables shared by all of them, a CertSecure-authored
`argument_spec.yml` file documents that playbook's variable contract --
name, type, required/secret flags, default, description, and (for constrained
values) `choices`. This is **CertSecure's own format**, not Ansible's
`meta/argument_specs.yml`: these playbooks are plain plays (`- hosts: all`),
not roles, so Ansible itself never reads this file. It exists purely so
`GET /aapIntegration/playbooks/files` can return a `variables` array per
playbook (merging the shared `_acme_lib/vars/argument_spec.yml` with the
playbook's own, the playbook's own winning on a name collision) for the
CertSecure frontend's Create Job Template / Launch Job dialogs to pre-fill
extra vars from. See `docs/design/acme-playbook-variables.md` for the prose
contract these files are authored from.

`argument_spec.yml` is resolved through the same three-tier lookup as
playbook content (disk `roles/` override > Mongo-saved edit > `roles_reference/`
baseline), but it is **not** itself a playbook or alias: it's excluded from
`GET /aapIntegration/playbooks/files`'s alias list and from
`build_playbooks_archive()`'s tarball, since AAP has no use for it. A
playbook with no `argument_spec.yml` at any tier (including a brand-new
custom playbook) simply reports `"variables": []` -- this is not an error.

A brand-new custom playbook can also get a real variable contract without
touching disk: `PUT /aapIntegration/playbooks/files/<alias>` accepts an
optional `variables` field alongside `content` --

```bash
PUT /aapIntegration/playbooks/files/my_app_deploy
{
  "content": "- hosts: all\n...",
  "variables": [
    {"name": "my_app_reload_command", "type": "str", "required": true,
     "description": "Reload command for my_app."}
  ]
}
```

-- saved as that playbook's own `argument_spec.yml` override (same Mongo tier,
same three-tier resolution as the content itself), then merged with the
shared `_acme_lib/vars/argument_spec.yml` exactly like a built-in playbook.
Omitting `variables` (every call before this existed) behaves exactly as
before -- it's purely additive to the existing `{"content": ...}` body. Same
list shape a `GET` already returns (`name`/`type`/`required`/`default`/
`secret`/`description`/`choices`), and each entry needs at least a `name`.
`DELETE /aapIntegration/playbooks/files/<alias>` clears both the content and
argument-spec overrides together, so reverting a playbook doesn't leave a
stale variable contract behind.

### Certificate renewal playbooks (ACME)

One playbook per supported target, all following the same two-flow model (see
`docs/design/acme-playbook-variables.md` for the full variable contract):

| Playbook | Target |
|---|---|
| `nginx_deploy/main.yml` | nginx |
| `apache_deploy/main.yml` | Apache httpd |
| `iis_deploy/main.yml` | IIS (Windows) |
| `tomcat_deploy/main.yml` | Tomcat |
| `mongodb_deploy/main.yml` | MongoDB |
| `mssql_deploy/main.yml` | Microsoft SQL Server (Linux or Windows) |
| `oracle_deploy/main.yml` | Oracle Database (listener TLS) |
| `custom_deploy/main.yml` | anything else -- fully variable-driven, no baked-in app logic |
| `f5_deploy/main.yml` | F5 BIG-IP (client-SSL profile / virtual server, via iControl REST), single target |

Explicit multi-target variants of every target above -- same app logic as the
single-target version, but the targets are supplied entirely via a
`deploy_targets` extraVar instead of already being registered as hosts in
AAP's Inventory (see `docs/design/acme-playbook-variables.md`'s "Explicit
multi-server playbooks" section for the full model). F5 gets one even though
`f5_deploy` never used AAP Inventory either (a BIG-IP has no SSH/WinRM
surface to register as an Inventory host) -- the multi variant is still for
renewing several virtual servers/profiles in one job run, on one BIG-IP or
several:

| Playbook | Target |
|---|---|
| `nginx_multi_deploy/main.yml` | nginx, explicit server list |
| `apache_multi_deploy/main.yml` | Apache httpd, explicit server list |
| `iis_multi_deploy/main.yml` | IIS (Windows), explicit server list |
| `tomcat_multi_deploy/main.yml` | Tomcat, explicit server list |
| `f5_multi_deploy/main.yml` | F5 BIG-IP, explicit list of virtual servers/profiles |

Two flows, selected per job/host via `acme_flow`:

- **`private`** (default) -- requests the certificate from CertSecure's own ACME
  server (`/v2/acme`), authenticated with an EAB kid5/hmac pair from a
  `CertSecure.ACME.Users` profile (`POST /acme/generate_profile`). CertSecure
  issues the certificate itself, so it's already in Inventory -- no separate
  enrollment step runs.
- **`public`** -- requests the certificate directly from a public CA's own ACME
  server (e.g. Let's Encrypt), using a `CertSecure.ACME.URL` entry's directory
  URL (and EAB, for CAs that require it) and DNS-01 validation via a
  `CertSecure.ACME.Domains` entry's provider credentials. Since CertSecure never
  saw this issuance, the playbook finishes by calling
  `POST /enrollment/enroll-acme-ca-cert` so the certificate still lands in
  CertSecure's Inventory -- recorded as a normal CA-issued certificate (same
  shape Generate Cert/Enroll CSR produce) under that public CA, not as a
  third-party import.

Linux targets use [acme.sh](https://github.com/acmesh-official/acme.sh);
Windows targets use [Posh-ACME](https://poshac.me/) -- both support EAB and a
wide range of DNS-01 provider plugins, which is what `_acme_lib/vars/dns_provider_map.yml`
maps `CertSecure.ACME.Domains`' friendly `provider` names onto.
