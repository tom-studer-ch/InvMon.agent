# InvMon Agent Skills

## Skills

This repository contains a number of Claude Code skills. See folder .claude/skills. 

Start a Claude Code session in this directory and Claude will automatically enumerate/load the skills. 

If you're using a different LLM/Agent system, just ask them to re-organize the file structure according to their requirements.


## Documentation 

There's a documentation folder under `doc` for each of the skills here (typically a tutorial with screenshots).


## Running

* Copy the InvMon.agent repository to your computer (via zip file download or a git clone).
* Make sure TWS is running, connected to your IB account.
* Make sure InvMon is running with a running MCP server (requires paid InvMon plan, enable the server in your Portfolio Group Target settings).
* Start claude code in your InvMon.agent directory. 
* then follow the instructions found in the doc directory for the particular skill you want to run.

To run a skill in a loop, use the Claude Code command (example)

```
/loop 1h /rating-agent
```

