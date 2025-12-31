# Circular Prompting

A prompt engineering technique for AI agents that involves iteratively repeating prompts across multiple conversation threads to manage context windows effectively.

## Core Concept

Instead of maintaining a single long conversation thread with context compaction, Circular Prompting involves:
- Running a prompt with context
- Allowing the AI to make progress
- Starting a new thread when context reaches a threshold (e.g., 60%)
- Repeating the same prompt to continue work
- Looping until the task is complete

## Use Cases

- **Project-specific**: Code remediation, refactoring, bug fixing with project context
- **Evergreen**: Code quality reviews, legacy code detection, general codebase analysis

## Advantages

- Avoids context degradation from conversation compaction
- Maintains fresh context in each iteration
- Clear success criteria (AI reports "no work left to do")
- Can be automated via a planning agent

## Example

```
Iteration 1: Prompt + Context → Progress → Stop at 60% context
Iteration 2: Same Prompt + Context → More Progress → Stop at 60% context
Iteration 3: Same Prompt + Context → Complete → "No work left to do"
```

## Documentation

See [`idea/notes/`](idea/notes/) for detailed transcripts and analysis of the concept.
