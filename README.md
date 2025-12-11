# Mastra Tool Search Processor Example

This example demonstrates how to implement a basic **Tool Search Processor** in Mastra. Instead of providing all tools to an agent upfront, this pattern allows the agent to dynamically discover and add tools during a conversation.

## Why Use a Tool Search Processor?

When agents have access to many tools, it can:

- Increase token usage (tool definitions are sent with every request)
- Overwhelm the model with too many options
- Lead to slower response times

A Tool Search Processor solves this by:

1. Starting conversations with minimal tools (just `tool-lookup` and `tool-add`)
2. Letting the agent discover available tools on demand
3. Adding only the tools needed for the current task

## How It Works

The `ToolSearchProcessor` implements the `Processor` interface and hooks into the `processInputStep` lifecycle:

```typescript
export class ToolSearchProcessor implements Processor {
  allTools: Record<string, Tool<any>> = {};      // All available tools
  enabledTools: Record<string, Tool<any>> = {};  // Tools added to conversation

  async processInputStep({ tools, messageList }: ProcessInputStepArgs) {
    // Inject system message explaining tool discovery
    messageList.addSystem(
      "To check available tools, call the tool-lookup tool..."
    );

    return {
      tools: {
        toolLookup: /* lists all available tools */,
        toolAdd: /* adds a tool to the conversation */,
        ...tools, /* pass through tools that were added to the agent or from other processors */
        ...this.enabledTools, /* add tools that have been enabled for the current conversation */
      },
    };
  }
}
```

### Meta-Tools Provided

| Tool          | Description                                                           |
| ------------- | --------------------------------------------------------------------- |
| `tool-lookup` | Returns a list of all available tools with their IDs and descriptions |
| `tool-add`    | Adds a specific tool to the current conversation by ID                |

## Project Structure

```
src/mastra/
├── index.ts                         # Mastra instance configuration
├── agents/
│   └── weather-agent.ts             # Example agent using the processor
├── processors/
│   └── tool-search-processor.ts     # The tool search processor implementation
└── tools/
    └── weather-tool.ts              # Example weather tool
```

## Usage

### 1. Create the Processor

```typescript
import { ToolSearchProcessor } from "./processors/tool-search-processor";
import { myTool1, myTool2, myTool3 } from "./tools";

const processor = new ToolSearchProcessor({
  tools: { myTool1, myTool2, myTool3 },
});
```

### 2. Attach to an Agent

```typescript
import { Agent } from "@mastra/core/agent";

export const myAgent = new Agent({
  id: "my-agent",
  name: "My Agent",
  model: "openai/gpt-4o",
  inputProcessors: [processor],
  // ... other config
});
```

### 3. Conversation Flow

When a user asks a question:

1. Agent receives the query with only `tool-lookup` and `tool-add` available
2. Agent calls `tool-lookup` to see what tools exist
3. Agent calls `tool-add` to enable the relevant tool(s)
4. Agent uses the newly added tool to answer the question

## Running the Example

```bash
# Install dependencies
pnpm install

# Start development server
pnpm dev
```

## Requirements

- Node.js >= 22.13.0
- pnpm

## Extending the Processor

You could enhance this pattern by:

- Adding semantic search to `tool-lookup` (filter tools by query)
- Implementing tool categories or tags
- Caching enabled tools across conversation turns
- Adding tool removal capability (e.g. `tool-remove`)
- Automatically removing tools from the conversation that haven't been called after N messages
- Persisting enabled tools to storage for continuity across server restarts
