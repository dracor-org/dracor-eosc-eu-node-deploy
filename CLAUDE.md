@AGENTS.md

## Claude Code specifics

* `.claude/settings.json` in this repository blocks reading and editing
  `~/.dracor-eosc-netrc` (the file with the admin password for agent-managed
  corpus loading, Step 6 b). Before offering that option, check that the
  block holds: a Read of the path and `cat` on it must be refused.
* Auto mode refuses some actions even when the person asks: writing to the
  instance through the MCP's write tools, and creating files that hold or
  template credentials. Do not try to work around it; tell the person what
  was refused and, if needed, give them the step to do themselves.
* Register the MCP with project scope (`.mcp.json` from `.mcp.json.example`,
  host filled in, trailing slash in `/mcp/`). Claude Code asks the person to
  approve it on the next start in this folder; the tools appear only then.
* Commands the person should run in this session start with `!`. Anything
  that involves typing the password goes into a separate terminal, never
  through `!`, since its input and output land in the conversation.
* For the clipboard on macOS use `pbcopy < local/<file>`.
