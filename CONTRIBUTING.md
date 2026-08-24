# Contributing

## Content ownership

Every concept has exactly **one owning skill** that holds its full explanation.
Any other skill that needs the concept gets at most a one-line gotcha plus a
pointer to the owner (`see fcode-<owner>`). When documenting a new platform
feature, write it up once in the owner and add pointers elsewhere — do not
repeat the explanation across skills.

| Concept | Owner |
|---|---|
| Platform model: processes, modules, execution context, variables, the inheritance model (processes, modules, variables, locales), datastore vs storage, versioning & alias model | `fcode-core-concepts` |
| Platform journey & support: access requests, development teams, app creation, OAuth app setup, demo companies, Dev/production marketplace experience, installations, the workspace-kinds map (`dev-` / `prod-` / `deploy-` / `base-app`), release & promotion flow, publication (DatoCMS, private apps), escalation channels, support-agent behavior | `fcode-ama` |
| CLI commands and flow, `metadata.json` / `team.json` field reference, webhook auth mechanics, the three variables files, `variables.meta.json`, inherited resources on disk, the `FACTORIAL_TOKEN` OAuth procedure, `version_tag` on URLs | `fcode-cli` |
| Language usage: runtime helpers, code snippets, logging, dependencies, return values | `fcode-javascript` / `fcode-python` |
| `parametersSchema.json` field types, widgets, validation | `fcode-json-schema` |
| Form embedding, themes, access restriction, pre-render / pre-fill contract, multi-step | `fcode-forms` |
| Locales & translations: locale files and inheritance, the `fcode.i18n` helper, form-schema i18n tokens, execution-locale selection, locale versioning, `i18n:*` commands | `fcode-i18n` |
| Agent working method, MCP tools | `fcode-agent` |
| Pre-production app validation: the check catalog and its severities (Blocker/Warning/Suggestion), the `APP_VALIDATION_REPORT.md` format, the read-only review procedure and verdict rules — each checked *rule* stays with its owning skill | `fcode-code-validation` |
| Dev→prod code promotion: the `fcode remote:add` release flow, its validation gate, divergence and confirmation rules | `fcode-release` |
| Worked, adaptable examples (code, not rules — rules live with their owner) | `fcode-examples` |

One home per rule applies **within** a file too:

- **No "Edge cases" section that restates a Gotcha or a Procedure step.** If a
  case is worth an edge-case entry, it either belongs in the Gotchas or is
  already covered — don't say it twice in the same file.
- **No "Where did X go?" migration notes.** When the product moves or renames
  something, edit the prose to describe the current state; the move itself
  belongs in the commit message, not the skill.

## The JavaScript / Python twins

`fcode-javascript` and `fcode-python` are deliberate mirrors: the same
sections, the same semantics, language-specific idiom. **Every semantic change
must land in both files.** CI fails when their section headings diverge.

## Validation

CI validates every skill with
[`skills-ref`](https://github.com/agentskills/agentskills/tree/main/skills-ref)
on each pull request. Run it locally with:

```bash
for dir in skills/*/; do npx skills-ref validate "$dir"; done
```

## Frontmatter descriptions

The `description` field is the routing surface agents use to pick a skill.
Keep it to what the skill covers plus when to use it — a couple of sentences
with the trigger words that matter, not an exhaustive feature list. Don't
append keywords to it with every feature PR.
