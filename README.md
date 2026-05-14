# InvMon Agent Skill

## Setup 

* Create a working directory anywhere named 'invmon-agent'
* Copy .mcp.json into that directory
* Create sub-directories '.claude/skills/'
* Copy SKILL.md into that directory

## Running

* Make sure TWS is running
* Make sure InvMon is running with a running MCP server (requires InvMon PRO, enable the server in your Portfolio Group Target settings)
* Start claude code in your 'invmon-agent' directory. 
* then:

Use the command

```
/loop 1h /invmon-agent
```

to run the skill repeatedly.
