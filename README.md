# Factorial Code skills

Official [Agent Skills](https://agentskills.io/) for building on the
[Factorial Code](https://code.factorialhr.com) platform. These skills are the
canonical home of the Factorial Code AI rules — install them into any
skills-compatible coding agent (Claude Code, Codex, Cursor, …) so it knows how
to write processes and modules, use the CLI, author input-parameter schemas,
and embed forms.

## Install

Using [`skills`](https://github.com/vercel-labs/skills):

```bash
# Install all skills
npx skills add factorialco/factorial-code-skills

# Install a specific skill
npx skills add factorialco/factorial-code-skills --skill fcode-javascript
```

## Skills

| Skill | What it covers |
|-------|----------------|
| [`fcode-ama`](skills/fcode-ama) | The platform user journey and support: access requests, teams, app creation, OAuth setup, demo companies, the Test marketplace, releases, publication, escalation paths. |
| [`fcode-core-concepts`](skills/fcode-core-concepts) | Platform architecture: processes, modules, execution context, variables, datastore, storage, workspace layout. Start here. |
| [`fcode-javascript`](skills/fcode-javascript) | Writing JavaScript (Node.js v22) processes and modules. |
| [`fcode-python`](skills/fcode-python) | Writing Python 3.13 processes and modules. |
| [`fcode-json-schema`](skills/fcode-json-schema) | Authoring `parametersSchema.json` input-parameter schemas. |
| [`fcode-agent`](skills/fcode-agent) | The iterative, confirmation-driven workflow for building on Factorial Code end to end. |
| [`fcode-cli`](skills/fcode-cli) | Using the `fcode` CLI for local development and cloud sync. |
| [`fcode-forms`](skills/fcode-forms) | Embedding a process's input-parameter form on a webpage. |
| [`fcode-ui-triggers`](skills/fcode-ui-triggers) | Surfacing an app process as a button inside the Factorial UI: `uiTrigger` settings, the result envelope, form-backed triggers. |
| [`fcode-i18n`](skills/fcode-i18n) | Workspace locales and translations: locale files, the `fcode.i18n` helper, translated form schemas, and the `i18n:*` CLI commands. |
| [`fcode-code-validation`](skills/fcode-code-validation) | Pre-production review of an app: clone the workspace read-only, run the static check catalog, and write `APP_VALIDATION_REPORT.md` with a ✅/❌ promotion verdict. |
| [`fcode-release`](skills/fcode-release) | Promote an app's code from its `dev-` workspace to its `prod-` workspace via `fcode remote:add`, gated by validation and explicit confirmation. |
| [`fcode-examples`](skills/fcode-examples) | Reference implementations: a marketplace payroll integration, a custom app with install/uninstall lifecycle, and utility processes. |

## Format

Each skill is a directory under `skills/` containing a `SKILL.md`
([Agent Skills spec](https://agentskills.io/specification)) plus any bundled
`references/` and `assets/`. Validate with
[`skills-ref`](https://github.com/agentskills/agentskills/tree/main/skills-ref):

```bash
npx skills-ref validate skills/fcode-javascript
```

CI validates every skill on each pull request. Content-ownership rules for
contributors are in [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE)
