# ralph-wiggum

> **⚠️ EXPERIMENTAL - USE AT YOUR OWN RISK**
>
> This package is highly experimental. The iterative nature of the Ralph Wiggum technique can result in **high token usage and significant costs**. Each iteration consumes tokens, and the agent may run multiple iterations before completing a task. Monitor your usage carefully and set appropriate `maxIterations` limits.

An iterative AI agent that implements the "Ralph Wiggum" technique - continuously running until a task is completed.

## Installation

```bash
npm install ralph-wiggum ai zod
```

## What is the Ralph Wiggum Technique?

Named after the lovably persistent character from *The Simpsons*, the Ralph Wiggum technique is an iterative approach that keeps running an AI agent until a task is evaluated as complete. Unlike a standard tool loop that stops when the model stops calling tools, Ralph keeps trying until the job is done.

```
┌─────────────────────────────────────────────────┐
│  ┌─────────────────────────────────┐            │
│  │  Inner tool loop                │            │
│  │  (LLM ↔ tools until no more)    │            │
│  └─────────────────────────────────┘            │
│              ↓                                  │
│  Evaluate: "Is the TASK complete?"              │
│              ↓                                  │
│       No? → Run another iteration               │
│       Yes? → Return final result                │
└─────────────────────────────────────────────────┘
```

## Usage

### Basic Example (Self-Judging)

```typescript
import { RalphLoopAgent } from 'ralph-wiggum';

const agent = new RalphLoopAgent({
  model: 'anthropic/claude-opus-4.5',
  instructions: 'You are a helpful coding assistant.',
  evaluator: {
    type: 'self-judge',
    prompt: 'Has the task been fully completed? Answer YES or NO.',
  },
  maxIterations: 5,
});

const { result, completedSuccessfully, totalIterations } = await agent.generate({
  prompt: 'Create a function that calculates fibonacci numbers',
});

console.log(`Completed: ${completedSuccessfully}`);
console.log(`Iterations: ${totalIterations}`);
console.log(result.text);
```

### With a Separate Judge Model

Use a cheaper/faster model to evaluate completion:

```typescript
import { RalphLoopAgent } from 'ralph-wiggum';

const agent = new RalphLoopAgent({
  model: 'anthropic/claude-opus-4.5',
  instructions: 'You are a coding assistant.',
  evaluator: {
    type: 'judge-model',
    model: 'anthropic/claude-sonnet-4-20250514', // Cheaper model for evaluation
    prompt: 'Is the coding task complete? YES or NO.',
  },
});

const result = await agent.generate({
  prompt: 'Build a todo list with add/remove/toggle functionality',
});
```

### With Custom Callback Evaluation

Full control over completion logic:

```typescript
import { RalphLoopAgent } from 'ralph-wiggum';

const agent = new RalphLoopAgent({
  model: 'anthropic/claude-opus-4.5',
  instructions: 'You are a coding assistant.',
  evaluator: {
    type: 'callback',
    fn: async (context) => {
      // Custom logic - check files, run tests, etc.
      if (context.result.text.includes('DONE')) {
        return { isComplete: true, reason: 'Agent signaled completion' };
      }

      if (context.iteration >= 3) {
        return {
          isComplete: false,
          feedback: 'Please wrap up and finalize your solution.',
        };
      }

      return { isComplete: false };
    },
  },
});
```

### With Tools

```typescript
import { RalphLoopAgent } from 'ralph-wiggum';
import { tool } from 'ai';
import { z } from 'zod';

const agent = new RalphLoopAgent({
  model: 'anthropic/claude-opus-4.5',
  instructions: 'You help users with file operations.',
  tools: {
    readFile: tool({
      description: 'Read a file from disk',
      parameters: z.object({ path: z.string() }),
      execute: async ({ path }) => {
        // Your implementation
        return { content: '...' };
      },
    }),
    writeFile: tool({
      description: 'Write content to a file',
      parameters: z.object({ path: z.string(), content: z.string() }),
      execute: async ({ path, content }) => {
        // Your implementation
        return { success: true };
      },
    }),
  },
  evaluator: { type: 'self-judge' },
  maxIterations: 10,
});
```

### Streaming

```typescript
const stream = await agent.stream({
  prompt: 'Build a calculator',
});

for await (const chunk of stream.textStream) {
  process.stdout.write(chunk);
}
```

Note: Streaming runs non-streaming iterations until evaluation passes or the final iteration, then streams that last iteration.

### Callbacks

Monitor progress with callbacks:

```typescript
const agent = new RalphLoopAgent({
  model: 'anthropic/claude-opus-4.5',
  evaluator: { type: 'self-judge' },
  onIterationFinish: ({ iteration, isComplete, feedback, reason }) => {
    console.log(`Iteration ${iteration}: ${isComplete ? 'COMPLETE' : 'CONTINUING'}`);
    if (feedback) console.log(`Feedback: ${feedback}`);
  },
  onFinish: ({ totalIterations, completedSuccessfully }) => {
    console.log(`Finished in ${totalIterations} iterations`);
    console.log(`Success: ${completedSuccessfully}`);
  },
});
```

## API Reference

### `RalphLoopAgent`

#### Constructor Options

| Option | Type | Required | Default | Description |
|--------|------|----------|---------|-------------|
| `model` | `LanguageModel` | ✅ | - | The AI model to use (AI Gateway string format) |
| `evaluator` | `RalphEvaluator` | ✅ | - | How to evaluate task completion |
| `instructions` | `string` | ❌ | - | System prompt for the agent |
| `tools` | `ToolSet` | ❌ | - | Tools the agent can use |
| `maxIterations` | `number` | ❌ | `10` | Maximum outer loop iterations |
| `stopWhen` | `StopCondition` | ❌ | `stepCountIs(20)` | Inner tool loop stop condition |
| `onIterationFinish` | `function` | ❌ | - | Called after each iteration |
| `onFinish` | `function` | ❌ | - | Called when all iterations complete |

#### Evaluator Types

**Self-Judge** - Same model evaluates completion:
```typescript
{ type: 'self-judge', prompt?: string }
```

**Judge Model** - Separate model evaluates completion:
```typescript
{ type: 'judge-model', model: LanguageModel, prompt?: string }
```

**Callback** - Custom function evaluates completion:
```typescript
{ type: 'callback', fn: (context) => boolean | RalphEvaluatorResult }
```

#### Methods

**`generate(options)`** - Non-streaming generation
- Returns: `RalphLoopAgentResult`

**`stream(options)`** - Streaming generation
- Returns: `StreamTextResult`

## License

Apache-2.0

