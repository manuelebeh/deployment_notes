# Contributing

Thanks for helping keep these guides accurate! Commands go out of date quickly (new Ubuntu releases, new database versions, renamed packages), so every fix counts.

## Reporting a problem

[Open an issue](https://github.com/manuelebeh/deployment_notes/issues/new/choose) and pick a template:

- **Outdated or broken command** — include the guide, the command, your Ubuntu version (`lsb_release -a`) and the exact error.
- **New guide request** — describe the stack or tool you'd like covered.

## Proposing a change

1. Fork the repo and create a branch.
2. Edit the Markdown files.
3. **Test the commands on a fresh VPS** (or a fresh Ubuntu VM/container) — guides must work when copy-pasted from top to bottom.
4. Open a pull request explaining what changed and on which Ubuntu version you tested.

The CI checks Markdown formatting and links on every pull request. To run the same checks locally:

```bash
npx markdownlint-cli2 "**/*.md"
```

## Writing style

- **English and French versions:** guides exist in English (root folders) and French (`fr/`). If you change a command in one, change it in the other too — or mention in the PR that the other language needs updating.
- **Target:** Ubuntu 24.04 and 26.04 LTS. Mention differences between versions in a small table or note.
- **Commands** are run by a non-root user with `sudo`. Write `sudo` explicitly.
- **Placeholders:** `yourdomain.com`, `api.yourdomain.com`, `your_vps_ip`, `deploy` (Linux user), `app_db`, `app_user`, `your-repo`. Use documentation IPs (`203.0.113.10`) for examples.
- **Security first:** never open database or Node.js ports to the whole internet; prefer SSH tunnels; never use the database superuser in an app.
- **Code blocks** always have a language (`bash`, `nginx`, `js`, `yaml`, `sql`, `conf`, `text`…).
- **One step = one explanation + one code block.** Say *why* when it's not obvious.
- **Versions:** when a guide pins a version, update the table in the README (*Versions used in these guides*) and the "Last reviewed" date.
- Long optional content goes in `<details>` blocks.

## Structure

```text
README.md            index (English)
server/              server setup, web servers, Cloudflare, monitoring
deploy/              deploying apps (VPS, Docker, CI/CD, Vercel, PaaS)
databases/           database installs + backups
fr/                  French translation, same structure
```
