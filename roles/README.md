# roles/ (disk-level override tier)

Empty by default. This directory is the **highest-priority** override tier when
CertSecure builds the archive AAP syncs from (`GET /aapIntegration/playbooks/archive`,
`build_playbooks_archive()` in `classes/functions/automation_platform.py`):

For each playbook file (named the same as its counterpart under
`roles_reference/`, e.g. `nginx_deploy/main.yml`), the archive is built by
resolving content in this order:

1. **`roles/<path>`** (this directory) -- a file physically placed here on
   disk. Manual/emergency override only; **not preserved across a CertSecure
   upgrade** (it ships inside the same build artifact as everything else
   under `static/`).
2. **MongoDB** (`AAPPlaybookOverrides` collection) -- what
   `PUT /aapIntegration/playbooks/files/<path>` saves when a playbook is
   edited through CertSecure. **Survives upgrades**, since Mongo data isn't
   touched by a redeploy.
3. **`roles_reference/<path>`** -- the shipped baseline. The one copy of each
   playbook that needs updating per CertSecure release; edits saved to Mongo
   automatically carry forward on top of whatever's here after an upgrade.

See `../README.md` for the full editing workflow and
`docs/design/acme-playbook-variables.md` for the ACME playbooks' variable
contract.
