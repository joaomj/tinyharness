# Inspiration

## Agents decisions log
- https://github.com/deepseek-ai/deepseek-harness/tree/master/.agents/notes
- like an ADR
- risk: staleness. The code is the doc? 

## Codex harness slow startup
- lots of sync calls which can be async
- lazy loading
- See: `/Users/joao/projects/tinyharness/docs/ideas/83bfd40a-e36b-48d5-8147-2eff1cb22e57.jpeg`

## Agentic harness in C
- Hax: `https://github.com/OleksandrChekhovskyi/hax/tree/master`
- Minimalist, TUI, high performance. I like these things.
- single binary
- uses very few RAM
- unix tooling (i love this!)
- inspectable
- very portable (unix, macos)
- read: `https://github.com/OleksandrChekhovskyi/hax/blob/master/docs/philosophy.md`
- My diff:
  - i plan to use mostly API calls for LLMs, not local models
  - i use a few MCPs, not much. Dont need a marketplace.
  - what if we did something similar but in python or rust?

## Agentic Harness with a rust core
- OMP: `https://github.com/can1357/oh-my-pi`

## My opinion (to be refined)
- what i value most: performance (speed, memory usage)
  - portability is second
- i am not bind by UI: a web-app or TUI, i dont care. I just want maximum performance and little to almost nothing setup.
  - this is the reason i find the idea behind opencode cool: an http server is the core; its not coupled with the client (be it a web app, tui, app, telegram bot or whatever)
- regarding python: Hermes Agent (from Nous Research) uses mostly python and is the most popular opensource agent out there.
  - its more like a general agent (like claude code or claude cowork), but it has 2 features i like: python as the core and easy mobile interfaces (telegram for me is sufficient). Coding using termius in iphone is a pain, because the screen is small.
- i plan to use this agent for personal use, not enterprise-grade or public large scale usage
- LLM providers: openai compatible (the industry pattern), anthropic compatible and openai subscription (oauth)
- Other compatibility:
  - skills (the standard pattern)
  - subagents (not in the MVP, but this should be available in the near future: see the concepts of "loop engineering"): 
    - `https://x.com/arscontexta/status/2023957499183829467`
    - `https://x.com/poteto/status/2069824386283319343`
    - `https://x.com/AnatoliKopadze/status/2080668775796314331`
    - i am not found of agentic loops and graphs because this seems to consume tokens like crazy for little return, but seeing so many skilled engineers using them means this needs a more deeper investigation.
  - memory systems/providers/interfaces
  - **IMPORTANT:** Pi's philosophy of extreme modularity is key here. I want to easily add/remove features to the agent without needing to change its core.
- Token savings:
  - Some harness implement better cache read/cache hit behavior. I dont know how, but codex is famous for this. Pi too (second to codex).
- I fear the wave of supply chain attacks happening in the node ecosystem right now.

## About Pi, Albatross or other harnesses:
- its not a requirement for me to FORK any of them. They are inspirations, each with something i would like to keep.
- Pi: modularity
- Albatross: rust
- Hax: minimalistic harness with very low memory usage
- Hermes: python, mobile interface (my second most used interface)
- Opencode: cliente-server architecture
  - this is my current harness of use, but its bothersome on cheap VPS, specially startup load and unbounded database growth (`https://github.com/anomalyco/opencode/issues/33356#issuecomment-5377574066`)
