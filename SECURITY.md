# Security Policy

This repository contains guidance documentation and Claude Code plugin
definitions (skills, plugin manifests) — no server, executable binary, or
service is run from it. Security concerns here are most likely to look
like:

- A skill's instructions that could be manipulated via prompt injection
  into doing something unintended.
- A licensing or attribution requirement that can be silently bypassed.
- Anything else that could cause a consumer of these plugins to act
  unsafely.

## Reporting a vulnerability

Please **do not** open a public GitHub issue for a security concern.

Instead, use GitHub's private vulnerability reporting for this
repository: go to the **Security** tab → **Report a vulnerability**.
This opens a private draft advisory visible only to the maintainer, so
the issue can be discussed and fixed before any public disclosure.

If you're unable to use that flow, you can instead email
todd@endlessorbit.com with details.

We'll acknowledge reports as promptly as we can and follow up once a fix
or mitigation is available.
