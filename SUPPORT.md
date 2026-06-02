# Support — DDT (Databricks Data Tools)

DDT is in a 30-day public beta. We want your bug reports and feature requests — they directly shape the product. Here is how to reach us.

## Where to get help

### 1. GitHub Issues (preferred)

Open an issue at **https://github.com/GVOrganization/ddt-tools/issues/new/choose** and pick a template:

- **Bug report** — something broke or behaved unexpectedly.
- **Feature request** — something you wish DDT did.

Searching [existing issues](https://github.com/GVOrganization/ddt-tools/issues) first helps avoid duplicates.

### 2. From inside the product

- **CLI:** run `ddt feedback` to send a bug report or feature request without leaving your terminal. Add `ddt feedback --contact you@example.com` if you'd like us to email you when a reported bug is fixed.
- **VS Code:** run **DDT: Report a Bug…** from the command palette.

These go to the same place the maintainers triage from. Note that what you type is sanitized before sending — see [PRIVACY.md](./PRIVACY.md).

### 3. Email

For anything that doesn't fit an issue (private reports, data-deletion requests, general questions): **sdt.ddt.tools@gmail.com**.

Data-subject / privacy requests (access, deletion, GDPR erasure) also go to this address — see [PRIVACY.md](./PRIVACY.md).

## What to include in a bug report

The bug-report template asks for these — having them ready speeds things up:

- DDT version (`ddt --version`, or the extension version from the VS Code Extensions panel)
- Your OS and, for the CLI, your Node.js version
- Your cloud (AWS / Azure / GCP) and auth method (PAT / OAuth M2M), if relevant
- What you did, what you expected, and what actually happened
- Any error output (with anything sensitive removed)

If you enabled automatic error reporting on first run, a crash may already have sent us a **sanitized** diagnostic. A GitHub issue with the surrounding context still helps us connect it to a real workflow.

## Response time during the beta

This is a solo-maintained beta. Expect a first response within **1–2 business days**. Complex issues may take longer to resolve, but we'll acknowledge yours in that window.

## Useful links

- Privacy policy: [PRIVACY.md](./PRIVACY.md)
- Release notes: [RELEASES.md](./RELEASES.md)
- Project home: [README.md](./README.md)
