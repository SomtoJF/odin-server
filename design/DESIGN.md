## Odin Code: Architecture

Odin Code will have Three Modes

1. Ask Mode: System prompt focused on research and answering questions. Will not have access to the `Todo Write`, `Write File`
2. Edit Mode
3. **Plan Mode:** Data goes into the planner. The planner comes up with a plan on how to gather the required

**All modes will use the same architecture**
`Message -> Planner -> Tool Call -> Planner`
This `Planner -> Tool Call` loop will continue until the problem is solved.

Each mode will have different system prompts so that they can perform their tasks most effectively. **Planner will use the user-specified model while every other thing will use a lesser model**

**Odin Code State**

```go
type State struct {
	AgentMode AgentMode
	Messages Message[]
	SubAgents SubAgent[] // Currently running subagent. Each agent kills itself after completion of it's task
	Context ContextItem[]
	CustomInstructions string // Data from the ODIN.md
	Config Config // includes configuration data specifying permissions for the agent. Should be in an odinconfig.json file
	MessagesMx sync.Mutex // since state is a shared resource we will need to use a mutex to prevent RW race conditions
	SubAgentsMx sync.Mutex //
	StateMx sync.Mutex // since state is a shared resource we will need to use a mutex to prevent RW race conditions
	StdinMx sync.Mx // Different concurrent processed might want to use the standard input at the same time. Therefore, it is important we implement mutex locks
}

type Message struct {
	Body string // The actual message sent by the user
	Todos Todo[] // Can be empty especially if the message was sent in ask mode where it doesnt have access to the TodoWrite tool
	ToolHistory ToolHistory[] // A list of tools which were called to answer the user's message
	Updates string[] // String array of realtime updates made to the CLI to inform the user what is going on. When execution is done, we empty the updates array to remove it from the UI.
}

type AgentMode string
const (
	AgentModeAskMode AgentMode = "ask_mode"
	AgentModePlanMode AgentMode = "plan_mode"
	EditMode AgentMode = "edit_mode"
)

type TodoStatus string
const (
	TodoStatusCompleted TodoStatus = "completed"
	TodoStatusPending TodoStatus = "pending"
	TodoStatusInProgress TodoStatus = "in_progress"
)

type Todo struct {
	Id uint
	Status TodoStatus // "pending" or "completed"
	Content string
}

type ToolHistory struct{
	ToolName string
	AffectedFiles string[]
	ToolUseDescription string // Short description of the change that the tool made
}

type SubAgent struct{
	Id uint
	Todos Todo[]
}

type Config struct {
	AllowedCommands string[]
	ForbiddenCommands string[]
}

type ContextItem struct {
	Content string
	FilePath *string // context might come from a command result
	SourceCommand *string // source command for the data if the data is a result from a cli command
}
```

**Helper Functions**

```go
func WriteMessageToState (message string, *state State) {
	// Lock state with the messages mutex
	*state.MessagesMx.lock()
	// Append new message to state Messages array
	newMessages := append(*state.Messages, Message{Body: message})
	// Replace state
	*state.Messages = newMessages
	*state.MessagesMx.Unlock()
	// return new state
}

func PublishState (state *State) {
// Publish state using redis
}
```

### Agent Design

```go
// /agents/agent.go
type AgentInterface interface {
	Kill() void // used specifically for subagents
	Execute()
}

func NewMainAgent() *MainAgent{}
func NewSubAgent() *SubAgent{}

// /agents/mainagent/agent.go
type MainAgent struct {
	Tools []Tool
	State *State
}

func (ma *Agent) Kill() {}
func (ma *Agent) Execute() {}


// /agents/subagent/agent.go
type SubAgent struct {
	Id uint
	Tools []Tool
	State *State
	ParentState *State
}

func (sa *Agent) Kill() {
	// Find current running subagent in parent state and remove it.
	return
}

func (sa *Agent) Execute() {
	// Call planner on latest Message
	// Planner calls tools and resumes iteration loop
	// When the problem has been solved, we end the execution and kill the agent
	return sa.Kill()
}


```

### Planner

This will be the brain of the entire system. It has three main tasks.

1. Figure out the next logical step to take
2. Identify when the problem has been solved to end the iteration (planner -> tool call) loop
   **Planner Input**

```go
type PlannerInput struct{
	LatestMessage Message // latest message sent from the user (the message being processed)
	AvailableTools ToolDescription[] // list of tools the planner is allowed to call
	Context ContextItem[] // context retrieved from state
	CustomInstructions string // Data from the ODIN.md
	Config Config
}

type ToolDescription struct{
	ToolName string
	ToolDescription string
	ToolInput map[string]interface{}
}

type ToolExecutionCallInput struct{
	ToolName string
	ToolInput map[string]interface{}
}

type PlannerOutput struct {
	Explanation string `json:explanation`
	TaskCompleted bool `json:taskCompleted`
	ExecuteTool ToolExecutionCallInput `json:executeTool` // Return multiple tool call requests ONLY when multiple pieces of independent information are required or requested. Or multiple independent actions need to be performed at the same time. Returning multiple tool execution calls i.e array with more than one element will cause the tools to be executed concurrently.
}
```

> > You have the capability to call multiple tools in a single response. When multiple independent pieces of information are requested, batch your tool calls together for optimal performance. When making multiple bash tool calls, you MUST send a single message with multiple tools calls to run the calls in parallel. For example, if you need to run "git status" and "git diff", send a single message with two tool calls to run the calls in parallel.

**TODO**: Figure out how to manage context
**TODO**: Figure out exact conditions to end iteration loop

### Tool Call

These are the various tools the planner can call
**Return Object**

```go
// /tools/tool.go
type Tool interface {
	PreHook(input map[string]interface{})
	Execute(prehookresponse any) // Fill the args out later
	PostHook(input map[string]interface{})
}

type ToolName string
const (
	ToolNameTodoWrite ToolName = "TodoWrite"
	// other tool names...
)

const TOOL_LIST = map[ToolName]Tool{
	ToolNameTodoWrite: todowrite.NewTool(...args),
	// other tools
}

func ExecuteTool (toolName ToolName, input map[string]interface{}) ToolResponse {
	tool := TOOL_LIST[toolName]

	response := tool.PreHook(input)
	executionResult := tool.Execute(respose)
	toolResponse := tool.PostHook(executionResult)
	return toolResponse
}


type ToolResponse struct{
	Data map[string]interface{} // can be any type. should be unmarshaled into the specific return type of the tool
	Description string // Short description of the change that the tool made
}
```

#### Available Tools

TODO Make sure tools have access to state

##### TodoWrite

This tool is used to create and manage a structured task list for your current coding session. This helps you track progress, organize complex tasks, and demonstrate thoroughness to the user.
It also helps the user understand the progress of the task and overall progress of their requests.

###### When to Use This Tool

Use this tool proactively in these scenarios:

1. Complex multi-step tasks - When a task requires 3 or more distinct steps or actions
2. Non-trivial and complex tasks - Tasks that require careful planning or multiple operations
3. User explicitly requests todo list - When the user directly asks you to use the todo list
4. User provides multiple tasks - When users provide a list of things to be done (numbered or comma-separated)
5. After receiving new instructions - Immediately capture user requirements as todos
6. When you start working on a task - Mark it as `in_progress` BEFORE beginning work. Ideally you should only have one todo as `in_progress` at a time
7. After completing a task - Mark it as completed and add any new follow-up tasks discovered during implementation

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "todos": {
      "type": "array",
      "description": "The updated todo list",
      "items": {
        "type": "object",
        "properties": {
          "id": {
            "type": "string"
          },
          "content": {
            "type": "string",
            "minLength": 1
          },
          "status": {
            "type": "string",
            "enum": ["pending", "in_progress", "completed"]
          }
        },
        "required": ["content", "status", "id"],
        "additionalProperties": false
      }
    }
  },
  "required": ["todos"],
  "additionalProperties": false
}
```

##### LS (LS Tool)

Lists files and directories in a given path. The path parameter must be an absolute path, not a relative path. You can optionally provide an array of glob patterns to ignore with the ignore parameter. You should generally prefer the Glob and Grep tools, if you know which directories to search.

**Input Schema**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "path": {
      "type": "string",
      "description": "The absolute path to the directory to list (must be absolute, not relative)"
    },
    "ignore": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "description": "List of glob patterns to ignore"
    }
  },
  "required": ["path"],
  "additionalProperties": false
}
```

##### Grep

A powerful search tool built on ripgrep
**Input Schema**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "pattern": {
      "type": "string",
      "description": "The regular expression pattern to search for in file contents"
    },
    "path": {
      "type": "string",
      "description": "File or directory to search in (rg PATH). Defaults to current working directory."
    },
    "glob": {
      "type": "string",
      "description": "Glob pattern to filter files (e.g. \"*.js\", \"*.{ts,tsx}\") - maps to rg --glob"
    },
    "type": {
      "type": "string",
      "description": "File type to search (rg --type). Common types: js, py, rust, go, java, etc."
    },
    "output_mode": {
      "type": "string",
      "enum": ["content", "files_with_matches", "count"],
      "description": "Output mode: \"content\" (lines), \"files_with_matches\" (paths), or \"count\" (totals)."
    },
    "-n": {
      "type": "boolean",
      "description": "Show line numbers (rg -n). Requires output_mode: \"content\"."
    },
    "-i": {
      "type": "boolean",
      "description": "Case insensitive search (rg -i)"
    },
    "-A": {
      "type": "number",
      "description": "Lines after match (rg -A). Requires output_mode: \"content\"."
    },
    "-B": {
      "type": "number",
      "description": "Lines before match (rg -B). Requires output_mode: \"content\"."
    },
    "-C": {
      "type": "number",
      "description": "Lines before/after match (rg -C). Requires output_mode: \"content\"."
    },
    "head_limit": {
      "type": "number",
      "description": "Limit output to first N lines/entries."
    },
    "multiline": {
      "type": "boolean",
      "description": "Enable multiline mode (rg -U). Default: false."
    }
  },
  "required": ["pattern"],
  "additionalProperties": false
}
```

##### Glob

Fast file pattern matching tool that works with any codebase size

- Supports glob patterns like "\*\*/\*.js" or "src/\*\*/\*.ts"
- Returns matching file paths sorted by modification time
- Use this tool when you need to find files by name patterns
- When you are doing an open ended search that may require multiple rounds of globbing and grepping, use the Agent tool instead

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "pattern": {
      "type": "string",
      "description": "The glob pattern to match files against"
    },
    "path": {
      "type": "string",
      "description": "The directory to search in. If not specified, the current working directory will be used. IMPORTANT: Omit this field to use the default directory. DO NOT enter \"undefined\" or \"null\" - simply omit it for the default behavior. Must be a valid directory path if provided."
    }
  },
  "required": ["pattern"],
  "additionalProperties": false
}
```

##### WriteFile

Overwrites an existing file in the filesystem.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "file_path": {
      "type": "string",
      "description": "The absolute path to the file to write (must be absolute, not relative)"
    },
    "content": {
      "type": "string",
      "description": "The content to write to the file"
    }
  },
  "required": ["file_path", "content"],
  "additionalProperties": false
}
```

##### ExecuteCommand

##### EditTool

Performs exact string replacements within a specific file. Planner must have already "read" the file to ensure the replacement string exists. We need to enforce the planner reads the file before, writing to it. So the tool should check the state ToolHistory to see if the last two entries involves reads to this specific file.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "file_path": {
      "type": "string",
      "description": "The absolute path to the file to modify"
    },
    "old_string": {
      "type": "string",
      "description": "The text to replace"
    },
    "new_string": {
      "type": "string",
      "description": "The text to replace it with (must be different from old_string)"
    },
    "replace_all": {
      "type": "boolean",
      "default": false,
      "description": "Replace all occurrences of old_string (default false)"
    }
  },
  "required": ["file_path", "old_string", "new_string"],
  "additionalProperties": false
}
```

##### MultiEditTool

A tool for making multiple edits to a single file in one operation. It is an atomic batch-processor for multiple changes. It is more efficient than calling the single `Edit` tool multiple times because it ensures all changes succeed together or none are applied at all.

**Critical Requirements:**

- **Atomicity:** If one edit fails (e.g., a string isn't found), the entire operation fails.
- **Sequential Logic:** Edits are applied in order. An early edit might change the text that a later edit is looking for—plan accordingly. (Might be benefifical to carry out the edits in a loop)
- **Strictness:** Requires absolute paths and exact whitespace matching.
- **File Creation:** Should not be able to create new files. To create new files, use the WriteFile tool

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "file_path": {
      "type": "string",
      "description": "The absolute path to the file to modify"
    },
    "edits": {
      "type": "array",
      "description": "Array of edit operations to perform sequentially on the file",
      "minItems": 1,
      "items": {
        "type": "object",
        "properties": {
          "old_string": {
            "type": "string",
            "description": "The text to replace"
          },
          "new_string": {
            "type": "string",
            "description": "The text to replace it with"
          },
          "replace_all": {
            "type": "boolean",
            "default": false,
            "description": "Replace all occurrences of old_string (default false)."
          }
        },
        "required": ["old_string", "new_string"],
        "additionalProperties": false
      }
    }
  },
  "required": ["file_path", "edits"],
  "additionalProperties": false
}
```

##### AgentTool

A subagent spawned by the main loop for running smaller complex tasks. Has it's own planner, tools and runs an iteration loop similar to the main loop. Doesn't have access to the Agent Tool i.e a subagent cannot spawn another subagent.
TODO: Figure out how to orchestrate

##### WebFetch Tool

Uses a crude go-colly scraper implementation to scrape web pages and converts the HTML to markdown (**Converting the HTML to markdown cleans up the data extremely effectively**). Then process it using a cheap AI model like claude haiku or gemini 2.5 flash
**Tool Input**

```go
type ToolInput struct {
	Url string // The url to scrape
	Prompt string // The prompt to run on the fetched content
}
```

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "url": {
      "type": "string",
      "format": "uri",
      "description": "The URL to fetch content from"
    },
    "prompt": {
      "type": "string",
      "description": "The prompt to run on the fetched content"
    }
  },
  "required": ["url", "prompt"],
  "additionalProperties": false
}
```

##### ContextSummarizer

##### InitTool

Looks through the codebase searching for data on the underlying architecture of the project and writes to the `ODIN.md` file. The `ODIN.md` file should contain information on:

1. A basic but detailed overview of what the project is i.e What we're trying to build.
2. The architecture of the project i.e Microservices, client server etc
3. The Components that make up the project and what tools they're written in.
4. Commands to run, build, test the project and other necessary commands.
   TODO Write detailed notes on how and where to retrieve this information from
   Input Object
   `no data`

### SafeGuards

- Any commands to be executed outside of the `pwd` will need explicit approval from the user. For example if the `ODIN.md` file is in the `/users/desktop/projects/odin` folder, any command targeted at any path not prefixed by the `ODIN.md` file path will need explicit user permission.

### Edge Cases and Considerations
