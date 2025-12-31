# Circular Prompting: Technical Architecture

## System Overview

Circular Prompting is a context management architecture for long-running AI agent tasks. It uses iterative prompt repetition across multiple conversation threads to maintain context freshness and avoid degradation from conversation compaction.

## Architecture Components

### 1. Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Planning Agent                           │
│  (Optional orchestrator for automated circular execution)   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Coordinates
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                   Context Manager                           │
│  - Maintains base context (project info, task definition)   │
│  - Monitors context window usage                           │
│  - Triggers thread restart at threshold                     │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Provides
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                 Prompt Template Engine                       │
│  - Stores the circular prompt (unchanged across iterations)│
│  - Injects base context into each iteration                 │
│  - Formats prompt for AI model                               │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Sends
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                  AI Model Interface                         │
│  - Claude Opus 4.5 (primary model)                          │
│  - Claude Code (code execution environment)                 │
│  - Manages conversation threads                              │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Executes
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                   Execution Engine                           │
│  - Applies AI-generated changes to codebase                  │
│  - Tracks progress and modifications                        │
│  - Reports completion status                                 │
└─────────────────────────────────────────────────────────────┘
```

### 2. Data Flow

```
┌──────────────┐
│ Base Context │
│ (Static)     │
└──────┬───────┘
       │
       │ + Circular Prompt
       ▼
┌─────────────────────────────────────────────────────────────┐
│                   Iteration N                               │
├─────────────────────────────────────────────────────────────┤
│ 1. Create new conversation thread                           │
│ 2. Inject: Base Context + Circular Prompt                   │
│ 3. Monitor context window usage                              │
│ 4. Allow AI to execute task                                 │
│ 5. Check: Context threshold reached?                        │
│    - No → Continue execution                                │
│    - Yes → Stop, save progress, prepare for next iteration  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Progress saved
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                   Termination Check                         │
├─────────────────────────────────────────────────────────────┤
│ AI Response Analysis:                                        │
│ - "No work left to do" / "Codebase clean" → SUCCESS        │
│ - Progress made, work remains → CONTINUE to N+1             │
│ - Errors/blockers → HANDLE and CONTINUE/RETRY               │
└─────────────────────────────────────────────────────────────┘
```

## Technical Specifications

### 3.1 Context Management

#### Base Context Structure

```yaml
base_context:
  project:
    name: "Home Inventory System"
    description: "Inventory management application"
    tech_stack:
      - "TypeScript"
      - "Node.js"
      - "Drizzle ORM"
      - "React"
  migration_history:
    from: "Goose ORM"
    to: "Drizzle ORM"
    completion: "95%"
    timestamp: "2025-12-31"
  task_objective:
    primary: "Remediate post-migration bugs"
    secondary: "Ensure robust, production-ready codebase"
  constraints:
    - "Maintain API compatibility"
    - "Preserve data integrity"
    - "Follow existing patterns"
```

#### Context Window Monitoring

```typescript
interface ContextMonitor {
  threshold: number;        // Default: 0.60 (60% of context window)
  currentUsage: number;     // Calculated from token count
  windowSize: number;       // Model's maximum context tokens
  model: "claude-opus-4.5";
  
  checkThreshold(): boolean {
    return this.currentUsage >= this.threshold;
  }
  
  triggerRestart(): void {
    // Signal planning agent to start new thread
  }
}
```

### 3.2 Circular Prompt Template

```markdown
# Task: Post-Migration Code Remediation

## Project Context
{{base_context}}

## Your Objective
You are remediating a codebase that recently migrated from Goose ORM to Drizzle ORM. The migration was 95% successful, but there are remaining bugs and inconsistencies that need to be addressed.

## Instructions
1. Review the codebase systematically
2. Identify and fix bugs related to the ORM migration
3. Ensure type safety and proper error handling
4. Maintain API compatibility
5. Follow existing code patterns and conventions
6. Report your progress after each significant change

## Success Criteria
- All migration-related bugs are resolved
- Codebase is robust and production-ready
- No legacy code or patterns remain
- All tests pass
- No further work is required

## Progress Reporting
After making progress, ask: "Do you want me to continue reviewing the codebase?"
```

### 3.3 Thread Lifecycle Management

```typescript
interface ThreadManager {
  currentThread: ConversationThread;
  iterationCount: number;
  progressTracker: ProgressState;
  
  startIteration(): void {
    this.currentThread = createNewThread();
    this.currentThread.inject(baseContext, circularPrompt);
    this.iterationCount++;
  }
  
  monitorExecution(): void {
    while (!contextMonitor.checkThreshold() && !isComplete()) {
      await this.currentThread.continue();
    }
    
    if (contextMonitor.checkThreshold()) {
      this.saveProgress();
      this.startIteration();
    }
  }
  
  saveProgress(): void {
    // Persist current state for next iteration
    // Track files modified, issues resolved
  }
}
```

### 3.4 Termination Detection

```typescript
interface TerminationAnalyzer {
  analyze(response: AIResponse): TerminationStatus {
    const completionIndicators = [
      /no work left to do/i,
      /codebase is clean/i,
      /no legacy code detected/i,
      /we're good/i,
      /task complete/i
    ];
    
    const progressIndicators = [
      /do you want me to continue/i,
      /shall i proceed/i,
      /more work to do/i
    ];
    
    if (completionIndicators.some(pattern => pattern.test(response.text))) {
      return TerminationStatus.SUCCESS;
    }
    
    if (progressIndicators.some(pattern => pattern.test(response.text))) {
      return TerminationStatus.CONTINUE;
    }
    
    return TerminationStatus.UNKNOWN;
  }
}
```

## Model Specifications

### Primary Model: Claude Opus 4.5

```yaml
model:
  name: "Claude Opus 4.5"
  provider: "Anthropic"
  capabilities:
    - "Advanced code understanding"
    - "Multi-file refactoring"
    - "Context-aware reasoning"
    - "Tool use and execution"
  context_window:
    max_tokens: 200000
    recommended_threshold: 0.60  # Restart at 60%
  strengths:
    - "Large-scale codebase analysis"
    - "Complex refactoring tasks"
    - "Maintaining consistency across changes"
```

### Execution Environment: Claude Code

```yaml
execution_environment:
  name: "Claude Code"
  capabilities:
    - "File system access"
    - "Code execution"
    - "Terminal commands"
    - "Git operations"
  integration:
    - "Direct file editing"
    - "Test execution"
    - "Build verification"
  features:
    - "Up-arrow prompt history"  # Easy prompt reuse
    - "Conversation management"
    - "Context monitoring"
```

## Execution Flow

### Manual Execution (Current Implementation)

```
User Action Flow:
1. User types circular prompt
2. Claude Code executes, makes progress
3. User monitors context window (via UI)
4. At ~60% context, user stops conversation
5. User creates new conversation
6. User presses ↑ to retrieve circular prompt
7. User resends prompt
8. Repeat until completion
```

### Automated Execution (Future Implementation)

```
Planning Agent Flow:
1. Initialize: Load base context, load circular prompt
2. Loop:
   a. Start new conversation thread
   b. Inject: base_context + circular_prompt
   c. Monitor context window usage in real-time
   d. When threshold (60%) reached:
      - Save current progress state
      - Terminate current thread
      - Increment iteration counter
   e. Analyze AI response for completion signals
   f. If complete → Exit loop, report success
   g. If not complete → Continue to next iteration
3. Finalize: Generate completion report
```

## State Management

### Iteration State

```typescript
interface IterationState {
  iterationNumber: number;
  startTime: ISO8601Timestamp;
  endTime: ISO8601Timestamp | null;
  filesModified: string[];
  issuesResolved: Issue[];
  issuesDiscovered: Issue[];
  contextPeakUsage: number;
  terminationReason: "threshold" | "completion" | "error";
}

interface ProgressState {
  totalIterations: number;
  completedIterations: IterationState[];
  currentIteration: IterationState | null;
  overallProgress: number;  // 0.0 to 1.0
  completionStatus: "in_progress" | "complete" | "blocked";
}
```

### Persistence Strategy

```yaml
persistence:
  location: ".circular-prompting/state.json"
  contents:
    - progress_state
    - iteration_history
    - base_context_hash
    - circular_prompt_hash
  recovery:
    - "Resume from last saved iteration"
    - "Verify base context hasn't changed"
    - "Replay skipped iterations if needed"
```

## Error Handling

### Error Scenarios

```typescript
enum ErrorType {
  CONTEXT_OVERFLOW,      // Exceeded context window before restart
  MODEL_RATE_LIMIT,      // API rate limit reached
  EXECUTION_FAILURE,     // Code execution failed
  INCONSISTENT_STATE,    // State corruption between iterations
  TERMINATION_AMBIGUITY  // Unable to determine completion
}

interface ErrorHandler {
  handle(error: Error): RecoveryAction {
    switch (error.type) {
      case ErrorType.CONTEXT_OVERFLOW:
        return RecoveryAction.IMMEDIATE_RESTART;
      case ErrorType.MODEL_RATE_LIMIT:
        return RecoveryAction.WAIT_AND_RETRY;
      case ErrorType.EXECUTION_FAILURE:
        return RecoveryAction.ROLLBACK_AND_RETRY;
      default:
        return RecoveryAction.MANUAL_INTERVENTION;
    }
  }
}
```

## Performance Considerations

### 3.8.1 Token Efficiency

```yaml
token_optimization:
  base_context_tokens: ~2000  # Static, injected each iteration
  prompt_template_tokens: ~500  # Static, injected each iteration
  ai_response_tokens: ~10000-50000  # Variable, per iteration
  total_per_iteration: ~12500-52500 tokens
  iterations_expected: 3-10  # For typical codebase remediation
  total_tokens_estimated: ~37500-525000 tokens
```

### 3.8.2 Time Efficiency

```yaml
time_estimates:
  iteration_duration: 5-15 minutes
  total_duration: 15-150 minutes (3-10 iterations)
  comparison_vs_compaction:
    - "Avoids context degradation"
    - "Slightly higher token cost (re-injecting context)"
    - "Potentially higher quality due to fresh context"
```

## Integration Points

### 3.9.1 Version Control

```typescript
interface GitIntegration {
  commitPerIteration: boolean;  // Optional: Commit after each iteration
  branchStrategy: "feature-branch" | "main-branch";
  commitMessageTemplate: string;
  
  createCommit(iteration: IterationState): void {
    const message = format(this.commitMessageTemplate, {
      iteration: iteration.iterationNumber,
      files: iteration.filesModified.length,
      issues: iteration.issuesResolved.length
    });
    
    git.commit(message, iteration.filesModified);
  }
}
```

### 3.9.2 Testing Integration

```typescript
interface TestRunner {
  runTests(iteration: IterationState): TestResult {
    const result = executeTestSuite();
    
    if (result.failed > 0) {
      iteration.issuesDiscovered.push(
        ...result.failures.map(f => ({
          type: "test_failure",
          file: f.file,
          description: f.message
        }))
      );
    }
    
    return result;
  }
}
```

## Monitoring and Observability

### 3.10.1 Metrics

```yaml
metrics:
  per_iteration:
    - context_window_usage
    - tokens_consumed
    - files_modified
    - issues_resolved
    - execution_duration
  overall:
    - total_iterations
    - total_tokens_consumed
    - total_execution_time
    - completion_rate
    - error_rate
```

### 3.10.2 Logging

```typescript
interface Logger {
  logIterationStart(iteration: number): void;
  logIterationEnd(iteration: number, state: IterationState): void;
  logContextThreshold(threshold: number, actual: number): void;
  logTermination(status: TerminationStatus): void;
  logError(error: Error): void;
}
```

## Security Considerations

```yaml
security:
  context_validation:
    - "Verify base context hasn't been tampered with"
    - "Sanitize file paths before execution"
  access_control:
    - "Restrict file system access to project directory"
    - "Validate all generated code before execution"
  audit_trail:
    - "Log all file modifications"
    - "Track iteration history for rollback"
```

## Future Enhancements

1. **Adaptive Threshold**: Dynamically adjust context threshold based on task complexity
2. **Context Summarization**: Summarize progress between iterations for continuity
3. **Parallel Iterations**: Run multiple independent circular prompts concurrently
4. **Progressive Context**: Gradually reduce base context as task progresses
5. **Multi-Model Support**: Switch between models based on task requirements
6. **Self-Optimizing Prompts**: Automatically refine circular prompt based on results

## Conclusion

The Circular Prompting architecture provides a robust framework for managing long-running AI tasks through strategic context management. By restarting conversations at optimal thresholds and maintaining fresh context, it addresses the limitations of conversation compaction while enabling systematic, iterative progress toward task completion.

The architecture is designed to be:
- **Modular**: Components can be swapped or enhanced independently
- **Observable**: Full visibility into execution state and progress
- **Resilient**: Error handling and recovery mechanisms built-in
- **Scalable**: Can be automated via planning agent or executed manually
- **Extensible**: Supports integration with testing, version control, and monitoring systems
