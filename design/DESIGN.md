## Odin Code: Architecture

Odin Code will have Three Modes

1. Ask Mode: System prompt focused on research and answering questions. Will not have access to any tools that can write to files (e.g., `TodoWrite`, `WriteFile`, `EditTool`, `MultiEditTool`, `InitTool`). Only read-only tools are available.
2. Edit Mode: Full access to all tools including file writing capabilities.
3. **Plan Mode:** Data goes into the planner. The planner comes up with a plan on how to gather the required information. Will not have access to any tools that can write to files (e.g., `TodoWrite`, `WriteFile`, `EditTool`, `MultiEditTool`, `InitTool`). Only read-only tools are available.

**All modes will use the same architecture**
`Message -> Planner -> Tool Call -> Planner`
This `Planner -> Tool Call` loop will continue until the problem is solved.

Each mode will have different system prompts so that they can perform their tasks most effectively. **Planner will use the user-specified model while every other thing will use a lesser model**

**Odin Code State**

```go
type State struct {
	AgentMode AgentMode
	Messages Message[]
	MessageQueue QueuedMessage[] // Queued messages when agent is busy
	IsExecuting bool // Tracks if agent is actively processing a message
	SubAgents SubAgent[] // Currently running subagent. Each agent kills itself after completion of it's task
	Context ContextItem[]
	CustomInstructions string // Data from the ODIN.md
	Config Config // includes configuration data specifying permissions for the agent. Should be in an odinconfig.json file

	// File Cache System - treats file data as key-value cache
	FileCache map[string]*CachedFile // filepath -> cached file data
	FileCacheConfig FileCacheConfig // Configuration for cache behavior

	// Mutexes for thread safety
	MessagesMx sync.Mutex // since state is a shared resource we will need to use a mutex to prevent RW race conditions
	MessageQueueMx sync.Mutex // mutex for message queue
	SubAgentsMx sync.Mutex //
	StateMx sync.Mutex // since state is a shared resource we will need to use a mutex to prevent RW race conditions
	StdinMx sync.Mx // Different concurrent processed might want to use the standard input at the same time. Therefore, it is important we implement mutex locks
	FileCacheMx sync.RWMutex // RWMutex for read-heavy file cache operations
}

type Message struct {
	Body string // The actual message sent by the user
	AnswerSummary string // Final answer/result returned to user after iteration loop completes
	Todos Todo[] // Can be empty especially if the message was sent in ask mode where it doesnt have access to the TodoWrite tool
	ToolHistory ToolHistory[] // A list of tools which were called to answer the user's message
	Updates string[] // String array of realtime updates made to the CLI to inform the user what is going on. When execution is done, we empty the updates array to remove it from the UI.
}

type QueuedMessage struct {
	Body string
	Mode AgentMode // Mode to use when processing this message
	Timestamp time.Time
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

// File Cache Types - implements efficient key-value cache for file data
// Cache structure mirrors actual read operations from tools
type CachedFile struct {
	FilePath     string    // Absolute file path (serves as the key)

	// Content storage - structure determined by actual reads, not file size
	FullContent  *string   // Full file contents (when tool reads entire file)
	PartialCache map[string]*CachedSegment // Line range segments (when tool reads specific ranges)
	IsPartial    bool      // True if PartialCache is used, false if FullContent

	// File metadata
	ContentHash  string    // SHA-256 hash for integrity checking (full file)
	Size         int64     // Total file size in bytes
	TotalLines   int       // Total number of lines in file

	// Timestamps for cache management
	CachedAt     time.Time // When file was first cached
	LastAccessed time.Time // For LRU eviction
	AccessCount  int       // Usage tracking
	ModTime      time.Time // File modification time from filesystem

	// State tracking
	IsModified   bool      // Track if file was written in this session
	OriginalHash string    // Hash at first read (for dirty detection)
}

// CachedSegment represents a cached portion of a file
// Created when tool reads specific line ranges (e.g., Grep with offset/limit)
type CachedSegment struct {
	StartLine    int       // Starting line number (1-based)
	EndLine      int       // Ending line number (inclusive)
	Content      string    // The actual content of this segment
	Hash         string    // Hash of this segment
	CachedAt     time.Time // When this segment was cached
}

type FileCacheConfig struct {
	MaxCacheSize     int64          // Total cache size in bytes (e.g., 100MB)
	MaxFileSize      int64          // Max size per file (e.g., 10MB)
	MaxEntries       int            // Max number of cached files (e.g., 500)
	EvictionPolicy   EvictionPolicy // LRU, LFU, or Hybrid
	TTL              time.Duration  // Optional: expire after duration
	EnableAutoRefresh bool          // Re-read if mod time changed

	// Partial caching settings - cache structure mirrors actual reads
	MaxSegments      int            // Max segments per file (e.g., 50)
}

type EvictionPolicy string
const (
	EvictionPolicyLRU    EvictionPolicy = "lru"    // Least Recently Used
	EvictionPolicyLFU    EvictionPolicy = "lfu"    // Least Frequently Used
	EvictionPolicyHybrid EvictionPolicy = "hybrid" // Prefer evicting unmodified files
)
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

// File Cache Helper Functions

func NewDefaultFileCacheConfig() FileCacheConfig {
	return FileCacheConfig{
		MaxCacheSize:      100 * 1024 * 1024,  // 100MB
		MaxFileSize:       10 * 1024 * 1024,   // 10MB per file
		MaxEntries:        500,                 // 500 files max
		EvictionPolicy:    EvictionPolicyHybrid, // Prefer evicting unmodified
		TTL:               0,                   // No expiration by default
		EnableAutoRefresh: true,                // Auto-detect file changes

		// Partial caching follows actual reads
		MaxSegments:       50,                  // Max 50 segments per file
	}
}

// Cache Eviction Strategies

// EvictLRU - Evicts least recently used file
func (s *State) EvictLRU() {
	s.FileCacheMx.Lock()
	defer s.FileCacheMx.Unlock()

	var oldestPath string
	var oldestTime time.Time = time.Now()

	for path, cached := range s.FileCache {
		if cached.LastAccessed.Before(oldestTime) {
			oldestTime = cached.LastAccessed
			oldestPath = path
		}
	}

	if oldestPath != "" {
		delete(s.FileCache, oldestPath)
	}
}

// EvictHybrid - Prefers evicting unmodified files, then uses LRU
func (s *State) EvictHybrid() {
	s.FileCacheMx.Lock()
	defer s.FileCacheMx.Unlock()

	var targetPath string
	var oldestUnmodified time.Time = time.Now()
	var oldestModified time.Time = time.Now()
	var hasUnmodified bool
	var oldestModifiedPath string

	// Find oldest unmodified and modified files
	for path, cached := range s.FileCache {
		if !cached.IsModified {
			hasUnmodified = true
			if cached.LastAccessed.Before(oldestUnmodified) {
				oldestUnmodified = cached.LastAccessed
				targetPath = path
			}
		} else {
			if cached.LastAccessed.Before(oldestModified) {
				oldestModified = cached.LastAccessed
				oldestModifiedPath = path
			}
		}
	}

	// Prefer evicting unmodified files
	if !hasUnmodified && oldestModifiedPath != "" {
		targetPath = oldestModifiedPath
	}

	if targetPath != "" {
		delete(s.FileCache, targetPath)
	}
}

// NeedsEviction - Checks if cache needs eviction before adding new file
func (s *State) NeedsEviction(newFileSize int64) bool {
	config := s.FileCacheConfig

	// Check entry count
	if len(s.FileCache) >= config.MaxEntries {
		return true
	}

	// Check total size
	currentSize := int64(0)
	for _, cached := range s.FileCache {
		currentSize += cached.Size
	}

	return currentSize + newFileSize > config.MaxCacheSize
}

// GetCachedFile - Thread-safe cache lookup
func (s *State) GetCachedFile(filePath string) (*CachedFile, bool) {
	s.FileCacheMx.RLock()
	defer s.FileCacheMx.RUnlock()

	cached, exists := s.FileCache[filePath]
	return cached, exists
}

// UpdateCacheAccess - Updates access tracking for cache entry
func (s *State) UpdateCacheAccess(filePath string) {
	s.FileCacheMx.Lock()
	defer s.FileCacheMx.Unlock()

	if cached, exists := s.FileCache[filePath]; exists {
		cached.LastAccessed = time.Now()
		cached.AccessCount++
		s.FileCache[filePath] = cached
	}
}

// GetCachedSegment - Retrieves a specific segment from partial cache
func (s *State) GetCachedSegment(filePath string, startLine, endLine int) (*CachedSegment, bool) {
	s.FileCacheMx.RLock()
	defer s.FileCacheMx.RUnlock()

	cached, exists := s.FileCache[filePath]
	if !exists || !cached.IsPartial {
		return nil, false
	}

	key := fmt.Sprintf("%d-%d", startLine, endLine)
	segment, found := cached.PartialCache[key]
	return segment, found
}

// AddCachedSegment - Adds a segment to partial cache
func (s *State) AddCachedSegment(filePath string, segment *CachedSegment) {
	s.FileCacheMx.Lock()
	defer s.FileCacheMx.Unlock()

	cached, exists := s.FileCache[filePath]
	if !exists {
		// Create new partial cache entry
		cached = &CachedFile{
			FilePath:     filePath,
			IsPartial:    true,
			PartialCache: make(map[string]*CachedSegment),
			CachedAt:     time.Now(),
			LastAccessed: time.Now(),
			AccessCount:  1,
		}
		s.FileCache[filePath] = cached
	}

	if cached.PartialCache == nil {
		cached.PartialCache = make(map[string]*CachedSegment)
	}

	// Add segment with range key
	key := fmt.Sprintf("%d-%d", segment.StartLine, segment.EndLine)
	cached.PartialCache[key] = segment

	// Evict oldest segments if over limit
	if len(cached.PartialCache) > s.FileCacheConfig.MaxSegments {
		s.evictOldestSegment(cached)
	}
}

// evictOldestSegment - Removes oldest segment from partial cache
func (s *State) evictOldestSegment(cached *CachedFile) {
	var oldestKey string
	var oldestTime time.Time = time.Now()

	for key, segment := range cached.PartialCache {
		if segment.CachedAt.Before(oldestTime) {
			oldestTime = segment.CachedAt
			oldestKey = key
		}
	}

	if oldestKey != "" {
		delete(cached.PartialCache, oldestKey)
	}
}

// ValidateCacheSufficiencyForEdit - Uses LLM to check if cached content is sufficient for edit
func (s *State) ValidateCacheSufficiencyForEdit(filePath, oldString, newString string) (bool, string, error) {
	s.FileCacheMx.RLock()
	cached, exists := s.FileCache[filePath]
	s.FileCacheMx.RUnlock()

	if !exists {
		return false, "File not in cache. Must read file before editing.", nil
	}

	// Gather all cached content (full or partial)
	var cachedContent string
	var segmentInfo []string

	if cached.IsPartial {
		// Collect all cached segments
		for key, segment := range cached.PartialCache {
			cachedContent += fmt.Sprintf("\n--- Lines %d-%d ---\n%s",
				segment.StartLine, segment.EndLine, segment.Content)
			segmentInfo = append(segmentInfo, key)
		}
	} else {
		// Full content available
		if cached.FullContent != nil {
			cachedContent = *cached.FullContent
		}
	}

	// Call LLM to validate sufficiency
	validationInput := CacheSufficiencyValidationInput{
		FilePath:      filePath,
		OldString:     oldString,
		NewString:     newString,
		CachedContent: cachedContent,
		IsPartial:     cached.IsPartial,
		SegmentKeys:   segmentInfo,
	}

	result := CallCacheSufficiencyValidator(validationInput)
	return result.IsSufficient, result.Explanation, nil
}

type CacheSufficiencyValidationInput struct {
	FilePath      string
	OldString     string
	NewString     string
	CachedContent string
	IsPartial     bool
	SegmentKeys   []string
}

type CacheSufficiencyValidationOutput struct {
	IsSufficient bool   // true if cache has enough info for the edit
	Explanation  string // explanation of decision
	SuggestedLineRanges []LineRange // if insufficient, which ranges to read
}

type LineRange struct {
	StartLine int
	EndLine   int
	Reason    string // why this range is needed
}

// CallCacheSufficiencyValidator - Calls cheap LLM to validate cache sufficiency
func CallCacheSufficiencyValidator(input CacheSufficiencyValidationInput) CacheSufficiencyValidationOutput {
	// Use cheap, fast model (e.g., Haiku, GPT-4o-mini)
	prompt := BuildCacheSufficiencyPrompt(input)

	// Call LLM with structured output
	response := CallLLM(prompt, "haiku") // Use cheapest model

	return ParseCacheSufficiencyResponse(response)
}

func BuildCacheSufficiencyPrompt(input CacheSufficiencyValidationInput) string {
	cacheType := "full file"
	if input.IsPartial {
		cacheType = fmt.Sprintf("partial (segments: %v)", input.SegmentKeys)
	}

	return fmt.Sprintf(`You are a cache sufficiency validator. Your job is to determine if cached file content contains enough information to perform a requested edit operation.

**File**: %s
**Cache Type**: %s
**Requested Edit**:
- Replace: "%s"
- With: "%s"

**Cached Content**:
%s

**Your Task**:
Analyze whether the cached content contains sufficient information to safely perform this edit. Consider:

1. Does the cached content include the old_string to be replaced?
2. If partial cache: Does it include the surrounding context needed to understand the change?
3. Are there multiple potential matches? If so, is there enough context to distinguish them?
4. Could this edit affect code/content outside the cached regions?

**Response Format** (JSON):
{
  "is_sufficient": true/false,
  "explanation": "Brief explanation of your decision",
  "suggested_line_ranges": [
    {"start_line": 100, "end_line": 150, "reason": "Contains the target string"}
  ]
}

**Guidelines**:
- If full file is cached: almost always sufficient unless old_string not found
- If partial cache: check if segments contain old_string AND enough context
- If old_string not found in any cached segment: not sufficient
- If found but near segment boundaries: suggest expanded range for safety
- Be conservative: when in doubt, request more context

Respond with JSON only.`, input.FilePath, cacheType, input.OldString, input.NewString, input.CachedContent)
}

func HandleIncomingMessage(state *State, body string, mode AgentMode) {
	state.StateMx.Lock()
	isExecuting := state.IsExecuting
	state.StateMx.Unlock()

	if isExecuting {
		// Agent is busy - add to queue
		state.MessageQueueMx.Lock()
		state.MessageQueue = append(state.MessageQueue, QueuedMessage{
			Body: body,
			Mode: mode,
			Timestamp: time.Now(),
		})
		state.MessageQueueMx.Unlock()
	} else {
		// Agent is idle - process immediately
		go ProcessMessage(state, body, mode)
	}
}

func ProcessMessage(state *State, body string, mode AgentMode) {
	// Mark as executing
	state.StateMx.Lock()
	state.IsExecuting = true
	state.AgentMode = mode
	state.StateMx.Unlock()

	// Add message to history
	message := Message{Body: body}
	state.MessagesMx.Lock()
	state.Messages = append(state.Messages, message)
	messageIndex := len(state.Messages) - 1
	state.MessagesMx.Unlock()

	// Get tools for this mode
	tools := GetModeTools(mode, false)

	// Run iteration loop until task completed
	taskCompleted := false
	for !taskCompleted {
		// Build cached files info for planner
		state.FileCacheMx.RLock()
		cachedFiles := make([]CachedFileInfo, 0, len(state.FileCache))
		for path, cached := range state.FileCache {
			cachedFiles = append(cachedFiles, CachedFileInfo{
				FilePath:    path,
				Size:        cached.Size,
				IsModified:  cached.IsModified,
				CachedAt:    cached.CachedAt,
				IsTruncated: cached.IsTruncated,
			})
		}
		state.FileCacheMx.RUnlock()

		plannerInput := PlannerInput{
			LatestMessage: state.Messages[messageIndex],
			AvailableTools: tools,
			Context: state.Context,
			CustomInstructions: state.CustomInstructions,
			Config: state.Config,
			CachedFiles: cachedFiles,
		}

		plannerOutput := CallPlanner(plannerInput)

		if plannerOutput.TaskCompleted {
			// Store final answer
			state.MessagesMx.Lock()
			state.Messages[messageIndex].AnswerSummary = plannerOutput.Explanation
			state.MessagesMx.Unlock()

			taskCompleted = true
		} else {
			ExecuteTool(plannerOutput.ExecuteTool.ToolName, plannerOutput.ExecuteTool.ToolInput)
		}
	}

	// Return result to user (via UI/API)
	ReturnAnswerToUser(state.Messages[messageIndex].AnswerSummary)

	// Mark as idle
	state.StateMx.Lock()
	state.IsExecuting = false
	state.StateMx.Unlock()

	// Process next message in queue if any
	ProcessNextMessageInQueue(state)
}

func ProcessNextMessageInQueue(state *State) {
	state.MessageQueueMx.Lock()

	if len(state.MessageQueue) == 0 {
		state.MessageQueueMx.Unlock()
		return // No more messages
	}

	// Dequeue first message
	nextMessage := state.MessageQueue[0]
	state.MessageQueue = state.MessageQueue[1:]
	state.MessageQueueMx.Unlock()

	// Process it
	go ProcessMessage(state, nextMessage.Body, nextMessage.Mode)
}

func GetModeTools(mode AgentMode, isSubAgent bool) []Tool {
	// Base tools available to all modes (read-only tools)
	baseTools := []Tool{
		TOOL_LIST[ToolNameLS],
		TOOL_LIST[ToolNameGrep],
		TOOL_LIST[ToolNameGlob],
		TOOL_LIST[ToolNameWebFetch],
		TOOL_LIST[ToolNameContextSummarizer],
		TOOL_LIST[ToolNameExecuteCommand],
        TOOL_LIST[ToolNameTodoWrite],
	}

	// Start with base tools
	availableTools := baseTools

	// Add mode-specific tools
	switch mode {
	case AgentModeAskMode, AgentModePlanMode:
		// Ask Mode and Plan Mode: Only read-only tools (baseTools already set)
		// No additional tools added
		// Only main agents can spawn subagents
		if !isSubAgent {
			availableTools = append(availableTools, TOOL_LIST[ToolNameAgentTool])
		}

	case EditMode:
		// Edit Mode: Add all writing tools
		availableTools = append(availableTools,
			TOOL_LIST[ToolNameWriteFile],
			TOOL_LIST[ToolNameEditTool],
			TOOL_LIST[ToolNameMultiEditTool],
			TOOL_LIST[ToolNameInitTool],
		)

		// Only main agents can spawn subagents
		if !isSubAgent {
			availableTools = append(availableTools, TOOL_LIST[ToolNameAgentTool])
		}
	}

	return availableTools
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
func NewSubAgent(mode AgentMode, parentState *State) *SubAgent{}

// /agents/mainagent/agent.go
type MainAgent struct {
	Tools []Tool // Set dynamically per message
	State *State
}

func NewMainAgent() *MainAgent {
	return &MainAgent{
		State: &State{},
	}
}

func (ma *MainAgent) Execute(body string, mode AgentMode) {
	ProcessMessage(ma.State, body, mode)
}

func (ma *Agent) Kill() {}


// /agents/subagent/agent.go
type SubAgent struct {
	Id uint
	Tools []Tool // Tools available to this subagent (determined by mode, excluding AgentTool)
	State *State
	ParentState *State
	Mode AgentMode // The mode this subagent was spawned in
}

func NewSubAgent(mode AgentMode, parentState *State) *SubAgent {
	// Subagents get tools based on their mode, but NEVER get AgentTool
	// Subagents under no circumstances can spawn other subagents
	tools := GetModeTools(mode, true) // isSubAgent = true excludes AgentTool
	return &SubAgent{
		Tools: tools,
		State: &State{},
		ParentState: parentState,
		Mode: mode,
	}
}

func (sa *Agent) Kill() {
	// Find current running subagent in parent state and remove it.
	return
}

func (sa *Agent) Execute() {
	// Call planner on latest Message
	// Planner calls tools and resumes iteration loop
	// When the problem has been solved, we end the execution and kill the agent
	// Note: Subagents cannot spawn other subagents - AgentTool is not in their tool list
	return sa.Kill()
}


```

### Planner

This will be the brain of the entire system. It has three main tasks.

1. Figure out the next logical step to take
2. Identify when the problem has been solved to end the iteration (planner -> tool call) loop
3. Leverage the FileCache to avoid redundant file reads and ensure current context
   **Planner Input**

```go
type PlannerInput struct{
	LatestMessage Message // latest message sent from the user (the message being processed)
	AvailableTools ToolDescription[] // list of tools the planner is allowed to call
	Context ContextItem[] // context retrieved from state
	CustomInstructions string // Data from the ODIN.md
	Config Config

	// File Cache Information - exposes cached files to planner
	CachedFiles []CachedFileInfo // List of files currently in cache
}

// CachedFileInfo provides cache metadata to planner
type CachedFileInfo struct {
	FilePath    string    // Absolute file path (the cache key)
	Size        int64     // File size in bytes
	IsModified  bool      // Whether file was modified in this session
	CachedAt    time.Time // When file was cached
	IsTruncated bool      // Whether content was truncated for size
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

#### Planner Cache Usage

The planner receives `CachedFiles` information in its input, allowing it to:

1. **Avoid Redundant Reads**: Check if a file is already cached before calling ReadFile
2. **Reference Cached Content**: Use cached file data without re-reading from disk
3. **Track Modified Files**: See which files have been edited (IsModified=true) in this session
4. **Ensure Freshness**: The cache automatically validates file freshness using ModTime

**Planner Instructions for Cache Usage:**

```
You have access to a file cache that tracks recently read files:
- Files listed in CachedFiles have already been read and their content is available
- Files with IsModified=true have been edited in this session
- The cache automatically handles freshness validation
- Grep tool automatically caches files when reading content (output_mode="content")
- Before editing any file, it MUST be in the cache (enforced by EditTool PreHook)

Available cached files:
{{range .CachedFiles}}
- {{.FilePath}} ({{.Size}} bytes, cached {{.CachedAt}}, modified={{.IsModified}})
{{end}}
```

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
	ToolNameLS ToolName = "LS"
	ToolNameGrep ToolName = "Grep"
	ToolNameGlob ToolName = "Glob"
	ToolNameWriteFile ToolName = "WriteFile"
	ToolNameEditTool ToolName = "EditTool"
	ToolNameMultiEditTool ToolName = "MultiEditTool"
	ToolNameTodoWrite ToolName = "TodoWrite"
	ToolNameAgentTool ToolName = "AgentTool"
	ToolNameWebFetch ToolName = "WebFetch"
	ToolNameContextSummarizer ToolName = "ContextSummarizer"
	ToolNameInitTool ToolName = "InitTool"
	ToolNameExecuteCommand ToolName = "ExecuteCommand"
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

Lists files and directories in a given path with cache awareness. The path parameter must be an absolute path, not a relative path. You can optionally provide an array of glob patterns to ignore with the ignore parameter. You should generally prefer the Glob and Grep tools, if you know which directories to search.

**Cache Integration:**

- Can indicate which listed files are already in FileCache
- Provides cache status in output (cached/uncached indicator)
- Does not automatically cache files (only lists them)

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

A powerful search tool built on ripgrep with integrated file caching. When reading file contents, Grep automatically caches files in the FileCache map using filepaths as keys, preventing redundant disk reads and ensuring content freshness.

**Cache Integration:**

- When `output_mode` is "content": caches file contents automatically
- Checks FileCache first before reading from disk (O(1) lookup by filepath)
- Validates cache freshness using file ModTime
- Updates cache with file content and metadata after reading
- Implements smart eviction when cache limits are exceeded
- Marks which files have been read for EditTool validation

**Partial Caching Based on Read Behavior:**

- Cache structure mirrors what was actually read by the tool
- Grep with `offset`/`limit` parameters → Partial cache (only those lines)
- Grep reading entire file → Full cache (complete content)
- Each segment is cached with its own key: "startLine-endLine"
- Maintains up to 50 segments per file (configurable)
- Context lines (-A/-B/-C) determine segment boundaries
- Example: grep with offset=100, limit=50 caches lines 100-150 as one segment

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

Fast file pattern matching tool that works with any codebase size with integrated caching

- Supports glob patterns like "\*\*/\*.js" or "src/\*\*/\*.ts"
- Returns matching file paths sorted by modification time
- Use this tool when you need to find files by name patterns
- When you are doing an open ended search that may require multiple rounds of globbing and grepping, use the Agent tool instead

**Cache Integration:**

- When returning file contents (optional feature): caches in FileCache
- Can check cache to identify which files have already been read
- Updates file metadata in cache when accessing files

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

Overwrites an existing file in the filesystem and updates the FileCache.

**Cache Behavior:**

- Creates or updates FileCache entry with new content
- Marks file as IsModified=true in cache
- Computes and stores new content hash
- Updates ModTime to reflect filesystem change

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

Performs exact string replacements within a specific file. Uses the FileCache with intelligent LLM-based validation to enforce that files must be read before editing, ensuring the planner always works with current file content.

**LLM-Based Cache Validation in PreHook:**

- PreHook checks if file exists in FileCache map (using filepath as key)
- If not cached: returns error requiring file to be read first (via Grep)
- If cached: uses cheap LLM (Haiku) to validate cache sufficiency:
  - Checks if cached content contains the old_string
  - For partial cache: validates surrounding context is sufficient
  - Determines if there's enough information to safely make the edit
  - If insufficient: suggests specific line ranges to read
- LLM provides intelligent validation vs rigid string matching
- Detects if file may have changed since cached (via hash comparison)

**PreHook Flow:**

```go
1. Check if file in cache → If not, error: "Must read file first"
2. Gather all cached content (full or partial segments)
3. Call ValidateCacheSufficiencyForEdit(filePath, oldString, newString)
4. LLM analyzes: "Does cache contain old_string with enough context?"
5. If insufficient: return error with suggested line ranges
6. If sufficient: proceed with edit
7. PostHook: Update cache with new content
```

**Partial Cache Handling:**

- For partially cached files: LLM checks if segments contain target with context
- If segment not cached or insufficient: requests specific line range
- After edit: converts partial cache to full cache (since file is modified)
- Ensures consistency by invalidating all partial segments after edit

**Benefits of LLM Validation:**

- Smarter than exact string matching
- Understands context requirements
- Can identify ambiguous edits requiring more context
- Suggests specific ranges to read when insufficient
- Very cheap (uses Haiku/mini model)

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

- **LLM-Based Cache Validation:** PreHook uses cheap LLM to validate cache sufficiency for ALL edits
  - Checks if cache contains all old_strings with sufficient context
  - For partial cache: validates all edit locations are covered
  - If any edit location is insufficiently cached: suggests expanded ranges
  - Validates edits won't interfere with each other based on cached content
- **Atomicity:** If one edit fails (e.g., a string isn't found), the entire operation fails.`
- **Sequential Logic:** Edits are applied in order. An early edit might change the text that a later edit is looking for—plan accordingly. (Might be benefifical to carry out the edits in a loop)
- **Strictness:** Requires absolute paths and exact whitespace matching.
- **File Creation:** Should not be able to create new files. To create new files, use the WriteFile tool
- **Cache Update:** Updates FileCache with final content after all edits succeed

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

A subagent spawned by the main agent for running smaller complex tasks. Has it's own planner, tools and runs an iteration loop similar to the main loop.

**Critical Restrictions:**

- **Only main agents can use AgentTool**: Subagents under no circumstances can spawn other subagents. The `GetModeTools` function automatically excludes `AgentTool` when `isSubAgent = true`.
- When a subagent is spawned, it receives only the tools available in the current mode (via `GetModeTools(mode, true)`), which explicitly excludes `AgentTool`.
- Subagents are spawned with a specific mode and inherit the tool restrictions of that mode, but never receive `AgentTool` regardless of mode.

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

### Message Queue & Dynamic Mode Switching

#### Message Queue System

Odin Code implements an automatic message queuing system that enables users to send multiple commands sequentially without waiting for each response. This improves workflow for automation and complex multi-step tasks.

**How It Works:**

1. **Idle State**: When the agent is not processing a message (`IsExecuting = false`), incoming messages are processed immediately via `HandleIncomingMessage()`.

2. **Busy State**: When the agent is actively processing a message (`IsExecuting = true`), new incoming messages are automatically added to `MessageQueue[]` for sequential processing.

3. **Sequential Processing**: Each message in the queue is processed completely—running a full iteration loop until `TaskCompleted = true`—before the next message is dequeued.

4. **Automatic Dequeuing**: After completing a message and returning the answer to the user, the system automatically checks `MessageQueue[]` and processes the next message if one exists.

**Key Components:**

- `MessageQueue []QueuedMessage`: Array that stores queued messages with their body, mode, and timestamp
- `IsExecuting bool`: Flag that tracks whether the agent is actively processing a message
- `MessageQueueMx sync.Mutex`: Mutex to prevent race conditions when accessing the queue
- `HandleIncomingMessage()`: Routes messages to queue or immediate processing based on `IsExecuting` state
- `ProcessMessage()`: Processes a single message completely through the full iteration loop
- `ProcessNextMessageInQueue()`: Dequeues and processes the next message after current message completes

#### Sequential Processing Flow

Each message is processed independently and completely:

1. Message received → Check `IsExecuting`
2. If busy → Add to `MessageQueue[]`, if idle → Start processing
3. Set `IsExecuting = true` and `AgentMode = mode`
4. Add message to `Messages[]` history
5. Get tools for the mode: `GetModeTools(mode, false)`
6. Run iteration loop: `Planner → Tool Call → Planner` until `TaskCompleted = true`
7. Store final answer in `Message.AnswerSummary`
8. Return answer to user via UI/API
9. Set `IsExecuting = false`
10. Check `MessageQueue[]` → If messages exist, dequeue and process next

**Benefits:**

- **No Interruptions**: No mid-execution message merging or complex state management
- **Clean Separation**: One message → One iteration loop → One answer
- **Rapid Input**: Users can type multiple commands quickly without waiting
- **Mode Flexibility**: Each queued message can specify a different mode
- **Simple Logic**: Straightforward queue-based architecture

#### Dynamic Mode Switching

The agent is not locked to a specific mode. Each message can specify its own mode, and the system dynamically adjusts:

**Mode per Message:**

- Each `QueuedMessage` stores its mode alongside the message body
- When processing a message, `ProcessMessage()` sets `state.AgentMode = mode`
- Tools are retrieved dynamically: `GetModeTools(mode, false)`
- Different modes available: `ask_mode`, `plan_mode`, `edit_mode`

**Mode-Specific Tool Access:**

- **Ask Mode & Plan Mode**: Only read-only tools (LS, Grep, Glob, WebFetch, ContextSummarizer, ExecuteCommand, TodoWrite)
- **Edit Mode**: All tools including file-writing tools (WriteFile, EditTool, MultiEditTool, InitTool)
- **Main Agent**: Can spawn subagents via AgentTool in any mode
- **SubAgent**: Cannot spawn subagents (AgentTool excluded when `isSubAgent = true`)

**Example Flow:**

```
User sends Message 1 (mode: ask_mode) → Processing...
User sends Message 2 (mode: edit_mode) → Queued
User sends Message 3 (mode: plan_mode) → Queued
Message 1 completes → Returns answer
Message 2 dequeued → Processes in edit_mode with writing tools
Message 2 completes → Returns answer
Message 3 dequeued → Processes in plan_mode with read-only tools
Message 3 completes → Returns answer
```

#### State Management

**IsExecuting Flag:**

- Set to `true` at the start of `ProcessMessage()`
- Set to `false` after message processing completes and answer is returned
- Used by `HandleIncomingMessage()` to determine routing (queue vs. immediate)
- Provides clear system state visibility

**Thread Safety:**

- All state mutations protected by appropriate mutexes
- `StateMx`: Protects `IsExecuting` and `AgentMode`
- `MessageQueueMx`: Protects `MessageQueue[]` array
- `MessagesMx`: Protects `Messages[]` array
- Prevents race conditions in concurrent processing

**Message History:**

- Each processed message stored in `Messages[]` with full context
- Includes: `Body`, `AnswerSummary`, `Todos[]`, `ToolHistory[]`, `Updates[]`
- Provides complete audit trail of all interactions

### SafeGuards

- Any commands to be executed outside of the `pwd` will need explicit approval from the user. For example if the `ODIN.md` file is in the `/users/desktop/projects/odin` folder, any command targeted at any path not prefixed by the `ODIN.md` file path will need explicit user permission.

### File Cache System Summary

The file cache system is integrated directly into existing tools rather than requiring a separate ReadFile tool, providing several key benefits:

**Integration Approach:**

- **Grep**: Automatically caches file contents when `output_mode="content"`
- **Glob/LS**: Can check cache status and optionally cache file metadata
- **EditTool/MultiEditTool**: Use LLM-based validation in PreHook to ensure cache sufficiency
- **WriteFile**: Updates cache after modifications
- **No separate ReadFile tool needed** - caching happens transparently

**Partial Caching Support:**

- **Read-Based Caching**: Cache structure mirrors actual tool read operations
- **Segment-Based Storage**: Tool reads specific line ranges → cached as segments
- **Full vs Partial**: Tool reads entire file → full cache; tool reads ranges → partial cache
- **Smart Segment Management**: Up to 50 segments per file with LRU eviction
- **Memory Efficient**: Only caches what was actually accessed, not entire files

**Key Benefits:**

1. **Prevents Redundant Reads**: Files are cached on first access via Grep, subsequent operations use cache
2. **O(1) Lookups**: Direct map access using filepath as key
3. **Intelligent Read-Before-Write**: EditTool PreHook uses LLM to validate cache sufficiency
4. **Context-Aware Validation**: LLM checks if cached segments have enough context for edits
5. **Tracks Modifications**: IsModified flag indicates files changed in current session
6. **Smart Eviction**: Hybrid strategy prefers evicting unmodified files over modified ones
7. **Automatic Freshness**: ModTime comparison detects external changes
8. **Memory Bounded**: Configurable limits (100MB default, 500 files max)
9. **Handles Large Files**: Partial caching mirrors actual line-range reads
10. **Cost-Effective**: Uses cheapest LLM (Haiku) for validation - negligible cost

**Example Flow (Small File):**

```
1. User: "Search for TODO in auth.go" (500KB file)
2. Grep reads auth.go → Caches full content
3. Cache state: {"/project/auth.go": {FullContent="...", IsModified=false}}
4. User: "Replace TODO with FIXME"
5. EditTool PreHook → Checks cache, finds auth.go ✓
6. EditTool executes → Updates cache with new content, IsModified=true
7. Future operations on auth.go use cached version (no disk read)
```

**Example Flow (Partial Read with Partial Cache):**

```
1. User: "Read lines 1000-1100 of logs.txt"
2. Grep called with offset=1000, limit=100 → Reads only those lines
3. Cache state: {"/project/logs.txt": {
     IsPartial=true,
     PartialCache={
       "1000-1100": {content: "lines 1000-1100...", hash: "xyz"}
     }
   }}
4. User: "Show me line 1050 with context" (lines 1045-1055)
5. Cache hit on segment "1000-1100" (contains 1045-1055) → No disk read
6. User: "Read lines 5000-5100"
7. Cache miss → Grep reads lines 5000-5100, adds segment "5000-5100"
8. Cache now has 2 segments: only 200 lines cached total
9. No assumptions based on file size - cache mirrors actual reads
```

**Example Flow (LLM Validation in EditTool):**

```
1. User: "Replace 'oldFunc()' with 'newFunc()' in service.go"
2. EditTool PreHook called
3. Check cache: service.go exists with partial cache (lines 50-150, 300-400)
4. Call ValidateCacheSufficiencyForEdit()
5. LLM receives:
   - old_string: "oldFunc()"
   - Cached segments: lines 50-150, 300-400
6. LLM analyzes and finds "oldFunc()" on line 75 (in cached segment 50-150)
7. LLM response: {
     "is_sufficient": true,
     "explanation": "Found 'oldFunc()' on line 75 within cached segment with sufficient context"
   }
8. Validation passes → EditTool proceeds with edit
9. File written, cache updated to full content (IsModified=true)

**Counter-example (Insufficient Cache):**
1. User: "Replace 'config.timeout' with 'config.requestTimeout'"
2. EditTool PreHook called
3. Check cache: config.go has partial cache (lines 1-50)
4. LLM analyzes: "config.timeout" not found in lines 1-50
5. LLM response: {
     "is_sufficient": false,
     "explanation": "String 'config.timeout' not found in cached segments",
     "suggested_line_ranges": [
       {"start_line": 1, "end_line": 200, "reason": "Need to search broader range"}
     ]
   }
6. PreHook fails with error: "Insufficient cache. Please read lines 1-200 first."
7. Planner sees error, calls Grep to read suggested range
8. Retry edit → Now succeeds
```

### Edge Cases and Considerations
