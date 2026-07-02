# Introduction to agent skills
- skills
  - enterprise, personal, project, plugin
  - md frontmatter: name, description, model, allowed-tools
  - folder with a SKILL.md 
  - can reference assets in same folder
  - keep under 500 lines
- hooks
  - in hooks/*.md
- Claude.md
- subagents
  - in agents/*.md
- MCP

---
# Building with the Claude API
- Claude API
	- chat.messages.create()
	- with chat.messages.stream()
- structured data
	- prefilled message + stop sequences

- prompt evaluation
	- eval dataset + run through grader + calculate average grade
	- eval dataset
		- task, format, solution criteria
	- grader:
		- model with system prompt with XML tags <task></task> <solution></solution>
		- also outputs strengths, weaknesses, reasoning

- prompt engineering
	- clear and direct
		- first line of prompt is most important
		- clear (simple language, explicit)
		- direct (instructive, action verbs)
	- specific
		- guidelines: section with bullet points
		- follow these steps: with sequential steps (complex problems only)
	- xml to separate sections, like multi-page strings formated into the system prompt
	- multi-shot prompting:
		- just providing examples in the system prompt
		- here are some examples:
			<sample_input>a</sample_input>
			<ideal_output>b</ideal_output>
			be careful of x:
			<sample_input>x</sample_input>
			<ideal_output>y</ideal_output>
		- use highly graded examples of your eval in the multi-shot


- tools
	- tool function + JSONschema of params
	- anthropic.messages.create(tools=[])
	- TextBlock, ToolUseBlock, ToolResultBlock (could have many tooluse/toolresult)
	- ToolUseBlock comes from the agent
	- ToolResultBlock comes from the user (python function is run on client)
	- while true loop with 'stop_reason' handling
	- parse tool name from ToolUseBlock in response
	- tool use in streaming:
		- "type": "input_json" in the ContentBlockDelta blocks
		- fine_grain=True: disable tool json validation, needs to check on client
	- builtin tools:
		- text editor (only schema builtin)
		- web search, allowed_domains


- rag
	- take large doc -> chunk it -> make relevance search mechanism to add to prompt
	- chunking is the most complex part
		- by size (chunks of fixed size, w or w/o overlap),
		  by structure (md headers),
		  by semantics (NLP models)
	- relevance search: cosine distance of the user query with all vectors of db
	- multi-index RAG:
		- multiple search mechanisms
		- combine cosine distance with BM25 word match
			- good if you have IDs or specific strings in corpus
		- 2 results weighted by RRF (Reciprocal Rank Fusion), its a weighted avg


- Claude
	- extended thinking - enable when evals are not working
		- thinking_budget=x  (max_tokens needs to be bigger)
	- images
		- max 100, max 5MB img size
		- prompting can improve a lot. clear and direct, guidelines, etc
		- multi-shot with other images as example
	- pdf
		- can look at images inside pdf
		- "citations": { "enabled": True }
			- get citationblock
			- can also be used on plaintext, not only pdf
	- caching
		- cheaper, faster, 1 hour TTL,
		- cache_control: type: ephemeral
		- set in textblock as a breakpoint at least at every 1024 tokens
		- put on tool schemas and system prompts
			- tools[-1]["cache_control"] = "ephemeral"
		- cache_creation_input_tokens and cache_read_input_tokens at result block
	- files API
		- upload file ahead of time
		- then refer with source: type: file, file_id: on future image block
	- code execution tool
		- managed by anthropic
		- docker containers have no network access
		- can use pre-uploaded files from files API with ContainerUploadBlock
		- flow:
			- file upload with files API
			- messages.create
				- user message asking for text
				- container upload: file upload id
				- code execution tool schema
			- result: text + code_execution_output with file id
	- agents vs workflows
		- workflow patterns: evaluator optimizer, parallelizator, chaining, routing
		- agents: tools must be generic, should have environment inspection tools
---
# Introduction to MCP
- MCP
	- can communicate over stdio, http, websocket
	- implements a set of tools for Claude to use
	- mcp server
		- just a class with each function as tools
		- mcp inspector with `mcp dev`
	- mcp client
		-part of the lib (mcp.client.stdio.stdio_client, mc.ClientSession)
		- very convoluted.
		- Usually wrapped in own class that does:
			- session.list_tools(), session.call_tool(), does cleanup
	- resources
		- tools are called by Claude, resources are called by you (app)
		- they have uri, like docs://docs/{doc_id} or docs://docs/
		- could be remote files, strings, or whatever app needs
		- canon example: @remote_file.md
		- return json, string, list
	- prompts
		- like skills, but defined on the server
		- can accept arguments, like "review pr {num}, be adhd friendly"
		- canon example: /slash-command
		- returns `messages` json (like msg history)
---
# Claude Code in Action
- Claude code
	- claude.md
		- /init generates the file
		- /memory comment -> updates claude.md
		- repo: claude.md, user: claude.local.md, global: ~/.claude/claude.md
	- basics
		- @ to reference files in chat
		- alt+enter for newlines
	- thinking and planning
		- plan mode = shift + tab
		- thinking level = /effort
		- plan = breadth, thinking = depth
	- /compact between tasks that have overlap
	- .Claude/commands/*.md to define slash commands
	- claude mcp add "name" "command"
	- /install-github-app -> setup github actions
		- uses api key, has mcp support,
		- can be referenced on yaml
		- can mention @claude on PRs, like devin
	- hooks:
		- defined inside .claude/settings.json or settings.local.json
		- prevent .env read
			- PreToolUse (or PostToolUse)
			- runs script that parses tool use JSON
		- run linter/typechecker after file edit
		- run unit tests after plan implementation
		- could spawn another claude from command line but very manual
		- claude code SDK: claude -p on cli, import on python or on js