# Factorial Code skills

Official [Agent Skills](https://agentskills.io/) for building on the
[Factorial Code](https://code.factorial.dev) platform. These skills are the
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
| [`fcode-core-concepts`](skills/fcode-core-concepts) | Platform architecture: processes, modules, execution context, variables, datastore, storage, workspace layout. Start here. |
| [`fcode-javascript`](skills/fcode-javascript) | Writing JavaScript (Node.js v22) processes and modules. |
| [`fcode-python`](skills/fcode-python) | Writing Python 3.13 processes and modules. |
| [`fcode-json-schema`](skills/fcode-json-schema) | Authoring `parametersSchema.json` input-parameter schemas. |
| [`fcode-agent`](skills/fcode-agent) | The iterative, confirmation-driven workflow for building on Factorial Code end to end. |
| [`fcode-cli`](skills/fcode-cli) | Using the `fcode` CLI for local development and cloud sync. |
| [`fcode-forms`](skills/fcode-forms) | Embedding a process's input-parameter form on a webpage. |

## Format

Each skill is a directory under `skills/` containing a `SKILL.md`
([Agent Skills spec](https://agentskills.io/specification)) plus any bundled
`references/` and `assets/`. Validate with
[`skills-ref`](https://github.com/agentskills/agentskills/tree/main/skills-ref):

```bash
npx skills-ref validate skills/fcode-javascript
```

## License

[MIT](LICENSE)
