# Your own DraCor on the EOSC EU Node

Deploy your own [DraCor](https://dracor.org) instance (API, website, MCP
server, corpora of your choice) on the Cloud Container Platform of the
[EOSC EU Node](https://open-science-cloud.ec.europa.eu/), guided step by step
by a coding agent.

You need an institutional login that gives you credits on the EU Node (the
Small tier, 2 vCPU and 8 GiB, is enough) and a coding agent such as
[Claude Code](https://claude.com/claude-code). You click in the platform's
web console; the agent prepares the files, explains each step and helps when
something goes wrong. It never sees your password. About 30 minutes, plus
the time to load corpora.

## Start

```sh
git clone https://github.com/dracor-org/dracor-eosc-eu-node-deploy
cd dracor-eosc-eu-node-deploy
claude
```

Then say hello. The agent follows [AGENTS.md](AGENTS.md); Claude Code reads
it through [CLAUDE.md](CLAUDE.md). Other agents that read `AGENTS.md` should
work as well, but were not tested.

Without an agent: the steps are in `AGENTS.md` too, written for the agent but
readable for people; [DEPLOYMENT-LOG.md](DEPLOYMENT-LOG.md) records the test
deployment in full.

## What is in here

| Path | Content |
|---|---|
| [AGENTS.md](AGENTS.md) | the guide the agent follows |
| `manifests/` | the platform objects, one file per component, and `dracor-all.yaml` for a single paste |
| `images/` | Dockerfiles of the adapted DraCor images (official images plus the changes needed for a random user ID) |
| `examples/` | templates for files you create yourself |
| [FINDINGS.md](FINDINGS.md) | what the test deployment showed: resources, effort, security spot checks |
| [DEPLOYMENT-LOG.md](DEPLOYMENT-LOG.md) | dated record of the test deployment |

## Status

Tested on 20 and 21 October 2026 on the EU-2 cluster of the EOSC EU Node,
Small tier, personal project, with DraCor API 1.4.1, metrics 1.5.3, frontend
2.7.1 and DraCor MCP 1.0.0-alpha.4. Platform details change; `AGENTS.md`
lists the assumptions the guide relies on.

Good for a demo, teaching or project instance with public data; see the
security notes at the end of `AGENTS.md`.

## Feedback

Tried it? Please tell us how it went, whether it worked or where it got
stuck: open an
[issue](https://github.com/dracor-org/dracor-eosc-eu-node-deploy/issues) or
post in the thread about this guide in the EOSC User Forum
(FORUM-THREAD-URL). At the end of a session the agent offers to draft a short
report for you. Every run on another account or cluster shows whether the
guide still matches the platform.

## Credits

Developed by [Ingo Börner](https://orcid.org/0000-0001-8294-2541)
(University of Potsdam) with the help of Claude Opus 5.5 (Anthropic).
Background: BLOG-POST-URL.

DraCor is developed by the DraCor team and contributors at
[dracor-org](https://github.com/dracor-org); the images here are built on
their official images.

## Funding

The DraCorOS project is funded as a cascading grant by OSCARS. The authors
acknowledge the OSCARS project, which has received funding from the European
Commission's Horizon Europe Research and Innovation programme under grant
agreement No. 101129751.

## License

[MIT](LICENSE)
