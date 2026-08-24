Yes. Your Unix analogy is directionally right, with one correction: classic Unix made many resources accessible through a small file descriptor interface, while Plan 9 pushed the idea much further by representing system resources through file systems and namespaces.[1] Penguin is applying a similar reduction of interfaces to the agent’s mutable state: “what is editable or recorded lives in files,” while computation remains in the SDK.[2] ([9p][1])

For the specific goal you describe, a software engineer whose harness should gradually conform to how he works, I would choose the file centric concept as the primary architecture.

Not “everything literally is a file,” though. I would use:

```text
small protected core
        +
everything adaptive is a file
        +
plugins only when new executable capability is required
```

That combination is materially better suited to personal worktools than either a pure extension architecture or an “everything is a plugin” architecture.

The reason becomes clearer if we distinguish what actually changes when a developer’s tools “learn his way of working.”

Most changes are not new computation. They are changes in policy and knowledge.

Suppose after six months the harness has learned things like:

```text
I normally use pytest before implementation.

For Python repos, inspect pyproject.toml first.

I prefer rg rather than grep.

When debugging Prudentia, inspect CloudWatch before changing code.

Never modify migration files without showing me the diff.

For large refactors, create a worktree.

I prefer uv for new Python projects.

This repository deploys through GitHub Actions.

When I say "benchmark", run these commands.

When this particular error occurs, check X before Y.
```

None of these justify a plugin.

They naturally want to become something like:

```text
~/.worktool/
├── PROFILE.md
├── RULES.md
├── preferences/
│   ├── python.md
│   ├── git.md
│   └── debugging.md
├── skills/
│   ├── aws-debugging/
│   │   └── SKILL.md
│   ├── benchmark/
│   │   └── SKILL.md
│   └── release/
│       └── SKILL.md
├── projects/
│   ├── prudentia.md
│   └── project-x.md
├── workflows/
│   ├── bugfix.md
│   ├── refactor.md
│   └── deploy.md
├── memory/
│   └── ...
└── traces/
    └── ...
```

And this is precisely where file centricity has a huge advantage.

The model can read it.

You can read it.

Git can diff it.

Git can version it.

`rg` can search it.

You can copy it between machines.

Another agent can inspect it.

A model can propose a patch to it.

You can roll back one bad adaptation.

There is no translation layer between “what the agent thinks it knows about my workflow” and “the representation I inspect.”

Penguin explicitly takes this approach. Its architecture puts agent behavior in `system_config.yaml`, `AGENTS.md`, and Skills, traces in append only JSONL, benchmarks and snapshots in files, and says that the file layer owns “everything editable and everything recorded.”[2] ([GitHub][2]) The resulting agent state is explicitly described as an entire behavior represented by editable files.[2] ([GitHub][2])

That is almost ideal for personal adaptation.

Consider the three philosophies in terms of your use case:

| Philosophy                    | Best at                                                | Personal adaptation |
| ----------------------------- | ------------------------------------------------------ | ------------------- |
| Small core + extensions       | Keeping the harness understandable                     | Good                |
| Everything is a plugin        | Reconfiguring executable architecture                  | Usually excessive   |
| Everything adaptive is a file | Accumulating preferences, skills, memory and workflows | Excellent           |

Pi's philosophy gets the first part right. Its official documentation explicitly says the core is intentionally small and customization happens through extensions, skills, prompt templates and other resources.[3] Extensions themselves can add tools and behavior, while Skills provide model readable workflows loaded on demand.[4][5] ([Pi Dev][3])

This gives Pi an important virtue for a personal harness: there is a stable thing underneath your customization.

DeepSeek/Cordis solves a harder problem. In DeepSeek Harness, even file access, LLM adapters and the agent loop can be plugins mounted into the Cordis context.[6] Cordis configuration determines the plugin tree and supports recomposition and HMR.[7] ([GitHub][4])

That makes sense if what evolves is this:

```text
today:

AgentLoopA
FilesystemA
MemoryA
SandboxA
ModelRouterA


six months later:

AgentLoopB
FilesystemB
MemoryB
SandboxC
ModelRouterD
```

But that is not the normal kind of learning that would make your work environment gradually become yours.

Most personalization should look more like:

```text
Day 1
RULES.md
50 lines

Day 100
RULES.md
120 lines

skills/
  debugging/
  code-review/
  aws/
  benchmarking/

projects/
  project-a/
  project-b/

memory/
  recurring-problems/
  decisions/
```

rather than:

```text
Day 100

replace AgentLoop implementation
rewire dependency graph
replace ToolRegistry
mount new SessionStore
hot-swap ContextManager
```

The latter may occasionally be useful, but if the harness continuously responds to ordinary workflow discoveries by generating executable plugins, you have converted personalization into software maintenance.

That is the key distinction.

A preference expressed as Markdown is cheap:

```markdown
## Testing

When modifying Python application logic:

1. Run the nearest existing tests first.
2. Add a failing regression test.
3. Implement the change.
4. Run the affected test suite.
```

A preference expressed as a plugin introduces code, an API dependency, lifecycle semantics, compatibility concerns and the possibility of runtime bugs.

If a model is going to modify itself autonomously, I would strongly prefer that most of those mutations occur in the former domain.

There is also an important gradient of risk:

```text
lowest risk

memory.md
preferences.md
workflow.md
SKILL.md

↓
configuration

config.yaml

↓
scripts

benchmark.sh
deploy.py

↓
extensions

extension.ts

↓
runtime architecture

agent-loop.ts
sandbox.ts
tool-runtime.ts

highest risk
```

A good adaptive harness should climb that ladder only when necessary.

This is where the Penguin model is particularly interesting. Its documented optimization process edits `AGENTS.md`, Skills and configuration, evaluates the modified agent, keeps the candidate only when its score improves, and otherwise rolls it back.[8] ([GitHub][5]) It snapshots agent state before optimization rounds.[9] ([GitHub][5])

For personal worktools, I would generalize that idea beyond benchmarks.

Imagine the harness accumulating traces of your interaction:

```text
You asked agent to fix bug
        ↓
agent tried A
        ↓
you corrected it:
"Always inspect logs first here"
        ↓
task succeeds
        ↓
reflection process notices correction
        ↓
proposes:

projects/prudentia/debugging.md

+ Before modifying application code,
+ inspect the relevant CloudWatch logs.
```

Then instead of silently modifying itself:

```text
Harness learned this preference from 4 sessions:

"Use rg instead of grep for repository searches."

Persist to preferences/shell.md?

diff ...
```

Eventually sufficiently innocuous things could be automatically accepted, while structural changes would still require approval.

That creates a very attractive feedback loop:

```text
          ┌───────────────┐
          │     work      │
          └───────┬───────┘
                  │
                  ▼
               traces
                  │
                  ▼
             reflection
                  │
          ┌───────┴───────┐
          │               │
        memory          skill
          │               │
          └───────┬───────┘
                  │
                  ▼
           better next run
                  │
                  └──────────────►
```

Notice that virtually the entire loop can operate over text files.

And this gets at the Unix connection you noticed.

The important Unix insight is not really that files are inherently special. It is that a small, uniform interface drastically increases composability.

Instead of every device requiring its own conceptual API, Unix famously made disparate I/O accessible through common operations. Plan 9 generalized this further: its authors explicitly describe resources as file systems and use namespaces to compose them.[1] ([9p][1])

For LLMs, we happen to have an additional reason that makes the abstraction unusually powerful:

```text
Unix:

many resources
      ↓
small common interface
read/write/open/close


LLM harness:

many forms of adaptive state
      ↓
small common interface
read/edit/create/search
```

And the second interface happens to be something coding models are already trained to use continuously.

There is a further consequence I think matters a lot: files make the engineer and the agent share the same state representation.

Compare a plugin based harness:

```text
Agent's learned behavior
        ↓
TypeScript object
        ↓
plugin registrations
        ↓
runtime state

Developer wants to inspect it
        ↓
understand source/API/runtime
```

with a file centric harness:

```text
Agent's learned behavior
        ↓
skills/debugging.md

Developer wants to inspect it
        ↓
cat skills/debugging.md
```

That property is extremely valuable for a tool intended to live with you for years.

It also avoids an increasingly important problem I would call personalization opacity.

Imagine asking your harness in 2028:

> Why do you always run `uv sync` before tests in this repository?

The ideal answer should correspond to an inspectable artifact:

```text
projects/foo/python.md:18

"Run uv sync before tests because this project
generates dependencies during synchronization."

Learned from:
traces/2027-11-04/...
```

You can inspect it, edit it or delete it.

That is substantially preferable to:

> Some extension subscribed to `before_task_start` and modifies the execution plan.

Plugins should instead represent genuine capabilities:

```text
GitHub integration
Jira integration
browser control
AWS API
database introspection
debugger protocol
language server
container runtime
```

Those actually involve executable behavior and therefore deserve executable modules.

So if I were designing the architecture from scratch for exactly your stated objective, it would be:

```text
┌─────────────────────────────────────────────┐
│ PROTECTED MICROKERNEL                       │
│                                             │
│ model invocation                            │
│ tool permissions                            │
│ filesystem boundaries                       │
│ process execution                           │
│ session protocol                            │
│ audit / rollback                            │
└───────────────────┬─────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│ CAPABILITY LAYER                            │
│                                             │
│ plugins / adapters                         │
│                                             │
│ git    github    shell    browser           │
│ aws    db        lsp      debugger          │
└───────────────────┬─────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│ ADAPTIVE FILESYSTEM                         │
│                                             │
│ identity.md                                 │
│ preferences/*.md                            │
│ projects/*.md                               │
│ skills/*/SKILL.md                           │
│ workflows/*.md                              │
│ memory/*.md                                 │
│ scripts/*                                   │
│ traces/*.jsonl                              │
│ evaluations/*                               │
└─────────────────────────────────────────────┘
```

The model should be allowed to evolve the bottom layer aggressively.

It should be allowed to create or modify scripts somewhat more cautiously.

It should require approval to change the capability layer.

And it should essentially never be able to modify the security and execution kernel during ordinary use.

This also means I would not actually adopt any of the three slogans literally.

My preferred slogan would be:

> **Everything adaptive is a file. Everything executable is a capability. The kernel stays small.**

That gives you the best idea from Penguin, the best idea from Pi, and only the part of Cordis that becomes useful once the harness genuinely needs runtime structural composition.

If I had to choose only one of the three concepts for a single software engineer, however, I would choose **“everything adaptive is a file.”** It aligns the persistence model with Git and Unix tooling, gives the LLM the representation it manipulates best, makes years of personalization inspectable rather than opaque, and makes self modification mostly a data transformation problem rather than a code generation problem.

“Everything is a plugin” strikes me as a better answer to **how do I build an indefinitely extensible agent platform?**

“Everything adaptive is a file” is a better answer to **how do I build a tool that slowly becomes my tool?**

Sources

[1] Pike, R., Presotto, D., Thompson, K., Trickey, H., Winterbottom, P., “The Use of Name Spaces in Plan 9,” 1993. Plan 9 from Bell Labs. [https://9p.io/sys/doc/names.html](https://9p.io/sys/doc/names.html) ([9p][1])

[2] PrismShadow, “PenguinHarness Architecture,” official project documentation, accessed August 13, 2026. [https://github.com/Prism-Shadow/penguin-harness/blob/main/packages/docs/content/architecture.en.md](https://github.com/Prism-Shadow/penguin-harness/blob/main/packages/docs/content/architecture.en.md) ([GitHub][2])

[3] Pi, “Pi Documentation,” official documentation, accessed August 13, 2026. [https://pi.dev/docs/latest](https://pi.dev/docs/latest) ([Pi Dev][3])

[4] Pi, “Extensions,” official documentation. [https://pi.dev/docs/latest/extensions](https://pi.dev/docs/latest/extensions) ([Pi Dev][6])

[5] Pi, “Skills,” official documentation. [https://pi.dev/docs/latest/skills](https://pi.dev/docs/latest/skills) ([Pi Dev][7])

[6] DeepSeek AI, “Cordis Tutorial,” DeepSeek Harness official repository, accessed August 13, 2026. [https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-tutorial/index.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-tutorial/index.md) ([GitHub][4])

[7] DeepSeek AI, “Composition and HMR,” DeepSeek Harness official repository. [https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-tutorial/06-composition-and-hmr.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-tutorial/06-composition-and-hmr.md) ([GitHub][8])

[8] PrismShadow, “Self-Improvement,” candidate optimization procedure. [https://github.com/Prism-Shadow/penguin-harness/blob/main/packages/docs/content/self-improvement.en.md](https://github.com/Prism-Shadow/penguin-harness/blob/main/packages/docs/content/self-improvement.en.md) ([GitHub][5])

[9] PrismShadow, “Self-Improvement,” snapshots and auditability. [https://github.com/Prism-Shadow/penguin-harness/blob/main/packages/docs/content/self-improvement.en.md](https://github.com/Prism-Shadow/penguin-harness/blob/main/packages/docs/content/self-improvement.en.md) ([GitHub][5])

[1]: https://9p.io/sys/doc/names.html?utm_source=chatgpt.com "The Use of Name Spaces in Plan 9"
[2]: https://github.com/Prism-Shadow/penguin-harness/blob/main/packages/docs/content/architecture.en.md "penguin-harness/packages/docs/content/architecture.en.md at main · Prism-Shadow/penguin-harness · GitHub"
[3]: https://pi.dev/docs/latest?utm_source=chatgpt.com "Pi Documentation"
[4]: https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-tutorial/index.md?utm_source=chatgpt.com "deepseek-harness/docs/cordis-tutorial/index.md at master"
[5]: https://github.com/Prism-Shadow/penguin-harness/blob/main/packages/docs/content/self-improvement.en.md "penguin-harness/packages/docs/content/self-improvement.en.md at main · Prism-Shadow/penguin-harness · GitHub"
[6]: https://pi.dev/docs/latest/extensions?utm_source=chatgpt.com "Extensions · Documentation"
[7]: https://pi.dev/docs/latest/skills?utm_source=chatgpt.com "Skills · Documentation"
[8]: https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-tutorial/06-composition-and-hmr.md?utm_source=chatgpt.com "deepseek-harness/docs/cordis-tutorial/06-composition-and ..."


