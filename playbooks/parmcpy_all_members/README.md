# Standalone PARMCPY All-Member Copy

This directory is a complete Ansible project for replacing the failed `PARMCPY` job `J0041212`.

## What It Fixes

The original JCL called `OCOPY` against an entire PDS/PDSE:

```text
OCOPY INDD(PARMLIB) OUTDD(USSOUT)
```

That produced `BPXF102E` because `OCOPY` requires a member name. This project instead:

1. Finds each source library with `ibm.ibm_zos_core.zos_find`.
2. Enumerates every member.
3. Fetches each member to a temporary controller directory.
4. Copies each member to its named USS destination.
5. Converts IBM-1047 to UTF-8 while staging, then back to IBM-1047 for USS.
6. Replaces existing files so shortened members do not leave stale trailing content.
7. Removes the temporary controller files after each source library.

## Contents

- `parmcpy_all_members.yml`: standalone playbook.
- `tasks/copy_library_members.yml`: reusable per-library workflow.
- `inventory.yml`: connection and z/OS Python/ZOAU placeholders.
- `requirements.yml`: IBM z/OS core collection dependency.
- `ansible.cfg`: local project configuration.

## Prerequisites

The controller needs Ansible and the IBM z/OS core collection. The managed z/OS system needs a supported Python installation and ZOAU, as required by the collection.

Install the collection from this directory:

```bash
ansible-galaxy collection install -r requirements.yml
```

Edit `inventory.yml` before running:

- `ansible_host`: BES2 hostname or address.
- `ansible_user`: z/OS user with catalog read access and USS write access.
- `PYZ`: Python installation path on z/OS.
- `PYZ_VERSION`: Python version used by ZOAU.
- `ZOAU`: ZOAU installation path on z/OS.
- `ZOAU_PYTHON_LIBRARY_PATH`: required when `zoautil_py` is installed outside the default Python path.
- `environment_vars`: adjust to the site’s Python and ZOAU layout.

Do not commit real credentials or private host details into this scratch project.

## Syntax Check

Run from this directory:

```bash
ansible-playbook parmcpy_all_members.yml --syntax-check
```

## Interactive Password Authentication

Key authentication is not required. Use `--ask-pass` to have Ansible prompt
interactively for the SSH password:

```bash
ansible-playbook zos_ping.yml --ask-pass
```

For the full member copy:

```bash
ansible-playbook parmcpy_all_members.yml --ask-pass
```

The password is read from the terminal and is not stored in `inventory.yml`.
Do not add `ansible_password` to this scratch inventory. If sudo or become is
introduced later, that requires a separate `--ask-become-pass` prompt.

## Current `zos_ping` Compatibility Issue

The installed controller uses Ansible core 2.21.1, while the installed
`ibm.ibm_zos_core` 2.0.0 `zos_ping` action plugin expects the older Ansible
`_configure_module` return contract. Running `zos_ping.yml` therefore fails
before the z/OS REXX check with:

```text
not enough values to unpack (expected 4, got 2)
```

Use the fallback while this version mismatch exists:

```bash
ansible-playbook zos_ping_fallback.yml --ask-pass
```

The fallback uses Ansible `raw` and checks the z/OS Python interpreter, ZOAU,
and `iconv`. It does not use the broken collection action plugin. The original
`zos_ping.yml` remains available after aligning Ansible and the collection,
for example by using an Ansible core release supported by the installed
collection or upgrading the collection to one that supports Ansible 2.21.

## Execute

Run from this directory after reviewing the source list and destinations:

```bash
ansible-playbook parmcpy_all_members.yml
```

The playbook changes the target USS directories. It does not submit the original JCL and it does not modify the source MVS libraries.

## Source Libraries

The default list contains the 15 source libraries that allocated successfully or were valid candidates in the job report. The failed `CPY4` source is intentionally absent:

```text
SYS1.BES1.PARMLIB
```

The report showed that this dataset did not exist on BES2. Add it only after verifying the correct dataset name and intended source image. For example, if confirmed by the operator/catalog query:

```yaml
      - dataset: SYS1.BES2.PARMLIB
        destination: /home/bmctyb/config/sys1/bes1/parmlib
```

Do not assume `SYS1.BES2.PARMLIB` is the correct replacement.

## Encoding Warning

The configured source libraries are text-oriented PARMLIB, CLIST, TCPPARMS, VTAMLST, and PROCLIB libraries. The playbook therefore uses IBM-1047 and UTF-8 conversion. Do not use this playbook unchanged for binary members; create a binary-copy variant with encoding conversion disabled or set according to the member format.

Destination directories are created with mode `0770`. Adjust the permission policy if the copied files must be readable by other service IDs.

## Failure Behavior

A missing source library fails the play before member copying begins for that source. A member fetch or copy failure causes Ansible to stop, leaving the controller staging directory for diagnostic inspection only if cleanup is not reached. Review the task output and rerun after correcting catalog, RACF, USS, or collection issues.
