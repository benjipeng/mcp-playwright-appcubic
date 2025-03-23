# MCP-Playwright Server Technical Guide

## Project Overview

MCP-Playwright-Server is a Model Context Protocol (MCP) server that enables Claude to interact with web browsers using Playwright. This is a fork of the original project, available at: https://github.com/benjipeng/mcp-playwright-appcubic

## Technical Architecture

### Core Components

The project follows a layered architecture:

1. **Protocol Layer**: Implements the Model Context Protocol using the MCP SDK
   - `src/index.ts`: Entry point that creates and configures the MCP server
   - `src/requestHandler.ts`: Sets up request handlers for the MCP server

2. **Tool Definition Layer**: Defines available tools and their schemas
   - `src/tools.ts`: Creates tool definitions with input schemas
   - Each tool has parameters defined in JSON schema format

3. **Tool Implementation Layer**: Implements the actual tool functionality
   - `src/tools/browser/`: Browser automation tools
   - `src/tools/api/`: API testing tools
   - `src/tools/common/`: Shared types and utilities

4. **Browser Control Layer**: Manages browser instances and provides core functionality
   - `src/toolHandler.ts`: Central handler that maintains browser state and dispatches tool calls

### Implementation Details

#### Browser State Management

The browser state is managed in `toolHandler.ts`:

```typescript
// Global state
let browser: Browser | undefined;
let page: Page | undefined;

// Reset browser state when needed
export function resetBrowserState() {
  browser = undefined;
  page = undefined;
}

// Ensure a browser instance is available
async function ensureBrowser(browserSettings?: BrowserSettings) {
  // Implementation details for launching and managing browser
}
```

Browser state is maintained across tool calls, but can be reset if connectivity issues occur.

#### Tool Class Hierarchy

Tools are implemented using a class hierarchy:

1. Base class: `BrowserToolBase` in `src/tools/browser/base.ts`
   - Provides common functionality and error handling
   - Implements `safeExecute()` for reliable browser operations

2. Specific tool classes (e.g., `NavigationTool` in `src/tools/browser/navigation.ts`)
   - Extend the base class
   - Implement the `execute()` method with tool-specific logic

Example of a tool implementation:

```typescript
// Navigation tool implementation
export class NavigationTool extends BrowserToolBase {
  async execute(args: any, context: ToolContext): Promise<ToolResponse> {
    // Check browser and page availability
    
    return this.safeExecute(context, async (page) => {
      try {
        await page.goto(args.url, {
          timeout: args.timeout || 30000,
          waitUntil: args.waitUntil || "load"
        });
        
        return createSuccessResponse(`Navigated to ${args.url}`);
      } catch (error) {
        // Error handling
      }
    });
  }
}
```

#### Function Flow

When Claude makes a tool call:

1. `handleToolCall()` in `toolHandler.ts` receives the request
2. Tool instances are initialized if needed
3. The appropriate tool handler is selected based on the tool name
4. For browser tools:
   - `ensureBrowser()` is called to ensure a browser instance exists
   - The tool's `execute()` method is called with arguments and context
   - The tool typically calls `safeExecute()` from the base class
   - The result is returned to Claude

## Local Development Setup

1. Clone your fork of the repository:
   ```bash
   git clone https://github.com/benjipeng/mcp-playwright-appcubic.git
   cd mcp-playwright-appcubic
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Build the code:
   ```bash
   npm run build
   npm link
   ```

4. Configure Claude Desktop for local development by modifying the `claude-desktop-config.json` file:
   ```json
   {
     "mcpServers": {
       "playwright": {
         "command": "npx",
         "args": [
           "--directory",
           "/path/to/your/mcp-playwright-appcubic",
           "run",
           "@executeautomation/playwright-mcp-server"
         ]
       }
     }
   }
   ```

   Note: Replace `/path/to/your/mcp-playwright-appcubic` with the absolute path to your project directory.

5. **Important**: After modifying the configuration, completely close Claude Desktop and restart it.

## Development Workflow

### Modifying Existing Tools

1. Locate the tool implementation in `src/tools/browser/` or `src/tools/api/`
2. Modify the `execute()` method with your changes
3. Build the project with `npm run build`
4. Test your changes with Claude Desktop

### Adding New Tools

1. Define the tool schema in `src/tools.ts`
2. Create a new tool class in the appropriate directory
3. Update tool exports in `src/tools/index.ts`
4. Register the tool handler in `toolHandler.ts`
5. Build and test your changes

## Debugging

When developing locally:

1. Run Claude Desktop from the terminal to see MCP server logs
2. Set `headless: false` in the navigation tool to see the browser UI
3. Use the `playwright_console_logs` tool to retrieve browser console output

## Understanding Tool Implementation

Let's examine how a specific tool is implemented:

### Navigation Example

1. The tool is defined in `src/tools.ts`:
   ```typescript
   {
     name: "playwright_navigate",
     description: "Navigate to a URL",
     inputSchema: {
       type: "object",
       properties: {
         url: { type: "string", description: "URL to navigate to" },
         // other properties
       },
       required: ["url"],
     },
   }
   ```

2. The implementation is in `src/tools/browser/navigation.ts`:
   ```typescript
   export class NavigationTool extends BrowserToolBase {
     async execute(args: any, context: ToolContext): Promise<ToolResponse> {
       // Implementation
     }
   }
   ```

3. The tool is registered in `toolHandler.ts`:
   ```typescript
   function initializeTools(server: any) {
     if (!navigationTool) navigationTool = new NavigationTool(server);
     // other tools
   }
   ```

4. When called, the tool:
   - Receives the `url` parameter from Claude
   - Verifies browser and page availability
   - Uses Playwright's `page.goto()` method to navigate
   - Returns a success or error response 