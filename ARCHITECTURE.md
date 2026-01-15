# Bolt.new Architecture Documentation

## Table of Contents
1. [Overview](#overview)
2. [Main Application Flow](#main-application-flow)
3. [Key Components](#key-components)
4. [AI Integration](#ai-integration)
5. [WebContainer Integration](#webcontainer-integration)
6. [State Management](#state-management)
7. [File Structure](#file-structure)
8. [Technology Stack](#technology-stack)
9. [Data Flow](#data-flow)
10. [Example Flow](#example-flow)

---

## Overview

Bolt.new is an AI-powered full-stack web development platform that runs entirely in the browser. It combines:
- **AI Code Generation** using Claude 3.5 Sonnet
- **In-Browser Runtime** powered by StackBlitz's WebContainers
- **Real-time Editor & Preview** for instant feedback
- **Complete Development Environment** without local setup

The key innovation is giving the AI **complete control** over the development environment, including filesystem, node server, package manager, terminal, and browser console.

---

## Main Application Flow

### User Prompt → Working Application Pipeline

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ User Input   │ --> │ LLM Stream   │ --> │ XML Parsing  │
└──────────────┘     └──────────────┘     └──────────────┘
                                                   │
                                                   ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Live Preview │ <-- │ State Update │ <-- │ Action Exec  │
└──────────────┘     └──────────────┘     └──────────────┘
```

### Detailed Steps:

#### 1. **User Message Submission** (`BaseChat.tsx`)
- User enters prompt in chat textarea
- `sendMessage()` callback is triggered
- System collects file modifications since last message
- File changes converted to unified diff format
- Diff prepended to user's message for context

#### 2. **API Call** (`routes/api.chat.ts`)
- Message sent to `/api/chat` endpoint via POST
- Server receives full message history as JSON
- Messages formatted for Claude API
- `streamText()` invoked with Claude 3.5 Sonnet model

#### 3. **LLM Processing** (`lib/.server/llm/`)
- **Model**: Claude 3.5 Sonnet (claude-3-5-sonnet-20240620)
- **Max tokens**: 8,192 per segment
- **System prompt** defines:
  - WebContainer constraints (no pip, no native binaries, etc.)
  - Available shell commands
  - `<boltArtifact>` XML format for responses
  - File modification format (diff syntax)
- Response streamed back as `text/plain`
- **Continuation handling**: If response hits token limit, sends `CONTINUE_PROMPT` to continue (max 2 segments)

#### 4. **Artifact Parsing** (`Chat.client.tsx` → `useMessageParser`)
- `StreamingMessageParser` parses XML tags in real-time:
  - `<boltArtifact id="..." title="...">` - Project container
  - `<boltAction type="file" filePath="...">` - File creation/update
  - `<boltAction type="shell">` - Shell command execution
- Parser emits callbacks: `onArtifactOpen`, `onActionOpen`, `onActionClose`
- Actions queued in workbench store

#### 5. **Action Execution** (`ActionRunner`)
- **Sequential execution** of queued actions
- **File actions**: 
  - Create directories recursively
  - Write file content to WebContainer filesystem
  - Binary files supported (base64 encoded)
- **Shell actions**: 
  - Execute via WebContainer shell (`jsh`)
  - Environment: `npm_config_yes: true` (auto-approve prompts)
  - Output streamed to terminal
  - Abortable via signal
- Status tracking: `pending` → `running` → `complete`/`failed`

#### 6. **Live Preview** (`Workbench`)
- WebContainer exposes preview port (typically 5173, 3000, etc.)
- Preview component renders iFrame pointing to preview URL
- Hot module reload on file changes
- File watcher detects changes and updates UI

---

## Key Components

### Frontend Architecture

#### **Chat Components** (`app/components/chat/`)

| Component | Purpose |
|-----------|---------|
| `Chat.client.tsx` | Main orchestrator; manages message state, streaming, artifacts |
| `BaseChat.tsx` | Chat UI: textarea, messages, example prompts, send button |
| `Messages.client.tsx` | Renders scrollable message list (user + assistant) |
| `AssistantMessage.tsx` | Displays AI responses with artifact cards |
| `UserMessage.tsx` | Displays user messages with markdown support |
| `Artifact.tsx` | Artifact card showing title, action list, expandable view |
| `CodeBlock.tsx` | Syntax-highlighted code blocks using Shiki |

#### **Workbench Components** (`app/components/workbench/`)

| Component | Purpose |
|-----------|---------|
| `Workbench.client.tsx` | Split-panel container: editor/preview/terminal |
| `EditorPanel.tsx` | CodeMirror editor with language support |
| `Preview.tsx` | iFrame displaying live preview of running app |
| `FileTree.tsx` | File browser with expand/collapse, file selection |
| `Terminal.tsx` | xterm.js terminal for command output |
| `FileBreadcrumb.tsx` | File path breadcrumb navigation |
| `PortDropdown.tsx` | Dropdown to select preview port when multiple servers |

#### **Runtime Components** (`app/lib/runtime/`)

| Component | Purpose |
|-----------|---------|
| `StreamingMessageParser` | Parses XML artifacts from LLM stream; emits callbacks |
| `ActionRunner` | Executes file/shell actions sequentially in WebContainer |

---

## AI Integration

### Model Configuration

```typescript
// lib/.server/llm/model.ts
Provider: Anthropic (@ai-sdk/anthropic)
Model: claude-3-5-sonnet-20240620
Max Tokens: 8,192
Max Response Segments: 2 (for continuation)
```

### Streaming Implementation

```typescript
// lib/.server/llm/stream-text.ts
import { streamText } from 'ai';
import { getAnthropicModel } from './model';

// Convert messages to AI SDK format
const coreMessages = convertToCoreMessages(messages);

// Stream response
const result = await streamText({
  model: getAnthropicModel(),
  system: getSystemPrompt(),
  messages: coreMessages,
  maxTokens: MAX_TOKENS,
});

// Return text/plain stream
return result.toTextStreamResponse();
```

### System Prompt Structure

The system prompt (~280 lines) defines:

1. **Role**: "You are Bolt, an expert AI assistant and exceptional senior software developer"
2. **Environment constraints**:
   - WebContainer limitations (no Python pip, no native binaries, no git)
   - Available commands: cat, chmod, cp, echo, hostname, node, python3, npm, etc.
   - File system: POSIX-like, starts empty
3. **Response format**:
   - Use `<boltArtifact>` tags to encapsulate projects
   - `<boltAction type="file">` for file operations
   - `<boltAction type="shell">` for commands
4. **Best practices**:
   - Always provide full file content (no "... rest of code" placeholders)
   - Install dependencies before running
   - Use appropriate dev servers (Vite, Next.js, etc.)
5. **User modifications handling**:
   - When user edits files, diff format provided in message
   - AI should respect user changes and build upon them

### Artifact Format

```xml
<boltArtifact id="unique-id" title="Project Name">
  <boltAction type="file" filePath="package.json">
    {
      "name": "my-app",
      "scripts": {
        "dev": "vite"
      }
    }
  </boltAction>
  
  <boltAction type="file" filePath="src/App.tsx">
    import React from 'react';
    
    export default function App() {
      return <div>Hello World</div>;
    }
  </boltAction>
  
  <boltAction type="shell">
    npm install
  </boltAction>
  
  <boltAction type="shell">
    npm run dev
  </boltAction>
</boltArtifact>
```

### Continuation Handling

When responses exceed token limits:

```typescript
// lib/.server/llm/switchable-stream.ts
if (finishReason === 'length') {
  // Continue from where we left off
  const continuationMessages = [
    ...messages,
    { role: 'assistant', content: currentContent },
    { role: 'user', content: CONTINUE_PROMPT }
  ];
  
  // Create new stream (up to MAX_RESPONSE_SEGMENTS)
  return streamText({ messages: continuationMessages, ... });
}
```

---

## WebContainer Integration

### Initialization

```typescript
// lib/webcontainer/index.ts
export async function getWebContainer(): Promise<WebContainer> {
  if (webcontainerInstance) {
    return webcontainerInstance;
  }
  
  webcontainerInstance = await WebContainer.boot({
    workdirName: 'project'
  });
  
  return webcontainerInstance;
}
```

- **Lazy loading**: Only boots when first artifact is created
- **Client-side only**: Check `!import.meta.env.SSR`
- **Vite HMR persistence**: Cached across hot reloads

### File System Operations

```typescript
// File watching (lib/stores/files.ts)
const watcher = await webcontainer.fs.watch('/project', { recursive: true });

for await (const event of watcher) {
  if (event.type === 'add_file') {
    // File created
  } else if (event.type === 'change') {
    // File modified
  } else if (event.type === 'remove_file') {
    // File deleted
  }
}

// Writing files (lib/runtime/action-runner.ts)
await webcontainer.fs.mkdir(dirname(filePath), { recursive: true });
await webcontainer.fs.writeFile(filePath, content, { encoding: 'utf-8' });

// Reading files (lib/stores/files.ts)
const content = await webcontainer.fs.readFile(filePath, 'utf-8');
```

### Command Execution

```typescript
// lib/runtime/action-runner.ts
const process = await webcontainer.spawn('jsh', ['-c', command], {
  env: { npm_config_yes: 'true' },
  terminal: { cols: 80, rows: 30 }
});

// Stream output
process.output.pipeTo(
  new WritableStream({
    write(chunk) {
      terminal.write(chunk);
    }
  })
);

// Wait for completion
const exitCode = await process.exit;
```

### Preview/Servers

```typescript
// lib/stores/previews.ts
webcontainer.on('server-ready', (port, url) => {
  previewsStore.addPreview(port, url);
});

// Preview component (app/components/workbench/Preview.tsx)
<iframe src={previewUrl} />
```

---

## State Management

Bolt uses **Nanostores** for reactive state management. Nanostores are atomic stores that emit updates to subscribers.

### Core Stores

#### **chatStore** (`lib/stores/chat.ts`)
```typescript
{
  started: boolean,     // Has chat started?
  aborted: boolean,     // Was chat aborted?
  showChat: boolean     // Is chat visible?
}
```

#### **workbenchStore** (`lib/stores/workbench.ts`)
```typescript
{
  artifacts: Map<string, ArtifactState>,  // All artifacts
  showWorkbench: boolean,                 // Workbench visibility
  currentView: 'code' | 'preview',        // Active view
  unsavedFiles: Set<string>               // Modified files
}

// ArtifactState
{
  id: string,
  title: string,
  closed: boolean,
  runner: ActionRunner  // Executes actions for this artifact
}
```

#### **filesStore** (`lib/stores/files.ts`)
```typescript
{
  files: Map<string, FileNode>,      // File tree structure
  selectedFile: string | null,        // Currently open file
  modifiedFiles: Map<string, string>  // Files changed by user (for diff)
}

// FileNode
{
  kind: 'file' | 'directory',
  path: string,
  content?: string | Uint8Array
}
```

#### **editorStore** (`lib/stores/editor.ts`)
```typescript
{
  documents: Map<string, EditorDocument>,
  selectedFile: string | null,
  currentDocument: EditorDocument | null
}

// EditorDocument
{
  filePath: string,
  value: string,
  scroll: { top: number, left: number }
}
```

#### **previewsStore** (`lib/stores/previews.ts`)
```typescript
{
  previews: Array<{ port: number, url: string }>,
  currentPreview: { port: number, url: string } | null
}
```

#### **terminalStore** (`lib/stores/terminal.ts`)
```typescript
{
  showTerminal: boolean,
  terminal: Terminal | null  // xterm.js instance
}
```

### Store Updates & Reactivity

```typescript
// Update store
workbenchStore.setKey('showWorkbench', true);

// Subscribe to changes (React)
import { useStore } from '@nanostores/react';
const workbench = useStore(workbenchStore);

// Subscribe to changes (vanilla)
workbenchStore.subscribe((value) => {
  console.log('Workbench updated:', value);
});
```

---

## File Structure

```
bolt.new/
│
├── app/                          # Application code
│   ├── components/               # React components
│   │   ├── chat/                 # Chat UI components
│   │   │   ├── Chat.client.tsx          # Main chat orchestrator
│   │   │   ├── BaseChat.tsx             # Chat layout & input
│   │   │   ├── Messages.client.tsx       # Message list renderer
│   │   │   ├── AssistantMessage.tsx      # AI message display
│   │   │   ├── UserMessage.tsx           # User message display
│   │   │   ├── Artifact.tsx             # Artifact card component
│   │   │   └── CodeBlock.tsx            # Syntax-highlighted code
│   │   │
│   │   ├── workbench/            # Editor & preview components
│   │   │   ├── Workbench.client.tsx     # Split-panel container
│   │   │   ├── EditorPanel.tsx          # CodeMirror editor
│   │   │   ├── Preview.tsx              # Live preview iFrame
│   │   │   ├── FileTree.tsx             # File browser
│   │   │   ├── Terminal.tsx             # xterm terminal
│   │   │   ├── FileBreadcrumb.tsx       # Path navigation
│   │   │   └── PortDropdown.tsx         # Server port selector
│   │   │
│   │   ├── header/               # Top navigation bar
│   │   ├── sidebar/              # Chat history sidebar
│   │   ├── editor/               # CodeMirror integration
│   │   └── ui/                   # Reusable UI elements
│   │
│   ├── lib/                      # Core libraries
│   │   ├── .server/              # Server-only code
│   │   │   └── llm/              # LLM integration
│   │   │       ├── stream-text.ts       # Anthropic streaming
│   │   │       ├── model.ts             # Model config
│   │   │       ├── prompts.ts           # System prompt
│   │   │       ├── constants.ts         # Token limits
│   │   │       ├── switchable-stream.ts # Continuation
│   │   │       └── api-key.ts           # API key handling
│   │   │
│   │   ├── stores/               # Nanostores state
│   │   │   ├── chat.ts                  # Chat state
│   │   │   ├── workbench.ts             # Workbench state
│   │   │   ├── editor.ts                # Editor state
│   │   │   ├── files.ts                 # File system state
│   │   │   ├── previews.ts              # Preview ports
│   │   │   ├── terminal.ts              # Terminal state
│   │   │   └── theme.ts                 # Theme state
│   │   │
│   │   ├── runtime/              # Runtime execution
│   │   │   ├── message-parser.ts        # XML artifact parser
│   │   │   └── action-runner.ts         # Action executor
│   │   │
│   │   ├── webcontainer/         # WebContainer wrapper
│   │   │   ├── index.ts                 # Init & access
│   │   │   └── auth.client.ts           # Auth handling
│   │   │
│   │   ├── hooks/                # React hooks
│   │   │   ├── useMessageParser.ts      # Parse artifacts
│   │   │   ├── usePromptEnhancer.ts     # Enhance prompts
│   │   │   └── ...
│   │   │
│   │   ├── persistence/          # Data persistence
│   │   │   ├── db.ts                    # IndexedDB ops
│   │   │   └── useChatHistory.ts        # Load/save chats
│   │   │
│   │   └── fetch/                # API utilities
│   │
│   ├── routes/                   # Remix routes
│   │   ├── _index.tsx                   # Home page (/)
│   │   ├── chat.$id.tsx                 # Chat by ID
│   │   └── api.chat.ts                  # Chat API endpoint
│   │
│   ├── types/                    # TypeScript types
│   │   ├── artifact.ts                  # Artifact types
│   │   ├── actions.ts                   # Action types
│   │   ├── terminal.ts                  # Terminal types
│   │   └── ...
│   │
│   ├── utils/                    # Utility functions
│   │   ├── constants.ts                 # App constants
│   │   ├── diff.ts                      # Diff computation
│   │   ├── logger.ts                    # Scoped logging
│   │   ├── markdown.ts                  # Markdown parsing
│   │   └── ...
│   │
│   ├── styles/                   # Global styles
│   ├── root.tsx                  # Root layout
│   ├── entry.client.tsx          # Client entry
│   └── entry.server.tsx          # Server entry (SSR)
│
├── functions/                    # Cloudflare Workers
│   └── api/
│
├── public/                       # Static assets
├── types/                        # Global types
├── package.json                  # Dependencies
├── vite.config.ts                # Vite configuration
├── wrangler.toml                 # Cloudflare config
├── tsconfig.json                 # TypeScript config
└── uno.config.ts                 # UnoCSS config
```

---

## Technology Stack

### Core Technologies

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Framework** | Remix | React-based full-stack framework with SSR |
| **Build Tool** | Vite | Fast development server and bundler |
| **UI Library** | React 18 | Component-based UI library |
| **Styling** | UnoCSS + SCSS | Atomic CSS + scoped styles |
| **State Management** | Nanostores | Lightweight reactive stores |
| **Code Editor** | CodeMirror 6 | Extensible code editor |
| **Terminal** | xterm.js | Terminal emulator in browser |
| **Runtime** | WebContainer | Browser-based Node.js environment |
| **AI Provider** | Anthropic | Claude 3.5 Sonnet LLM |
| **AI SDK** | Vercel AI SDK | Unified AI streaming interface |
| **Streaming** | Web Streams API | Real-time data streaming |
| **Storage** | IndexedDB | Client-side database for chat history |
| **Deployment** | Cloudflare Pages | Edge deployment platform |
| **Package Manager** | pnpm | Fast, disk-efficient package manager |

### Key Dependencies

```json
{
  "dependencies": {
    "@ai-sdk/anthropic": "^0.0.39",        // Anthropic AI integration
    "@webcontainer/api": "1.3.0",          // WebContainer runtime
    "@remix-run/cloudflare": "^2.10.2",    // Remix framework
    "@codemirror/...": "^6.x",             // CodeMirror editor
    "@xterm/xterm": "^5.5.0",              // Terminal emulator
    "ai": "^3.3.4",                        // Vercel AI SDK
    "nanostores": "^0.10.3",               // State management
    "react": "^18.2.0",                    // UI library
    "diff": "^5.2.0",                      // Diff generation
    "shiki": "^1.9.1"                      // Syntax highlighting
  }
}
```

---

## Data Flow

### Complete Request/Response Cycle

```
┌─────────────────────────────────────────────────────────────────┐
│                      USER INTERACTION LAYER                      │
│  BaseChat: User types prompt → Click send                        │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ sendMessage(prompt + diffs)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    COMMUNICATION LAYER                           │
│  POST /api/chat                                                  │
│  Body: { messages: [...history, newMessage] }                   │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ streamText()
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AI PROCESSING LAYER                           │
│  Anthropic API: Claude 3.5 Sonnet                                │
│  System Prompt + Message History → Generated Response            │
│  Stream: <boltArtifact>...</boltArtifact>                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ text/event-stream
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    PARSING LAYER                                 │
│  StreamingMessageParser                                          │
│  • Extracts <boltArtifact> tags                                 │
│  • Parses <boltAction type="file|shell">                        │
│  • Emits callbacks: onArtifactOpen, onActionOpen, onActionClose │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ callbacks
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    STATE UPDATE LAYER                            │
│  workbenchStore.addArtifact()                                    │
│  workbenchStore.addAction()                                      │
│  filesStore updates, editorStore updates                         │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ state change notifications
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    EXECUTION LAYER                               │
│  ActionRunner.runAction()                                        │
│  Sequential execution:                                           │
│  1. File actions → WebContainer.fs.writeFile()                   │
│  2. Shell actions → WebContainer.spawn('jsh')                    │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ filesystem changes, process output
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    WEBCONTAINER LAYER                            │
│  • File system updated                                           │
│  • npm install executes                                          │
│  • Dev server starts (e.g., Vite)                               │
│  • Server exposes port → previewsStore.addPreview()             │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ preview URL available
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    RENDERING LAYER                               │
│  Components re-render based on store updates:                    │
│  • FileTree: Shows new files                                     │
│  • EditorPanel: Displays file content                            │
│  • Preview: iFrame loads preview URL                             │
│  • Terminal: Shows command output                                │
│  • AssistantMessage: Displays artifact card with action list     │
└─────────────────────────────────────────────────────────────────┘
```

### Store Interaction Flow

```
User Edit File
      │
      ▼
EditorPanel (onChange)
      │
      ▼
editorStore.updateDocument()
      │
      ▼
filesStore.markModified()
      │
      ▼
workbenchStore.addUnsavedFile()
      │
      ▼
UI Updates (unsaved indicator)
      │
      ▼
On Next Message: Generate Diff
      │
      ▼
Send to AI (diff included in prompt)
```

---

## Example Flow

### Creating a React Todo App

#### 1. User Input
```
User: "Create a React todo app with TypeScript and Tailwind CSS"
```

#### 2. API Request
```http
POST /api/chat HTTP/1.1
Content-Type: application/json

{
  "messages": [
    {
      "role": "user",
      "content": "Create a React todo app with TypeScript and Tailwind CSS"
    }
  ]
}
```

#### 3. AI Response (Streamed)
```xml
I'll create a React todo app with TypeScript and Tailwind CSS using Vite!

<boltArtifact id="react-todo-app" title="React Todo App">
<boltAction type="file" filePath="package.json">
{
  "name": "react-todo-app",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "@vitejs/plugin-react": "^4.0.0",
    "autoprefixer": "^10.4.14",
    "postcss": "^8.4.24",
    "tailwindcss": "^3.3.2",
    "typescript": "^5.0.2",
    "vite": "^4.3.9"
  }
}
</boltAction>

<boltAction type="file" filePath="index.html">
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Todo App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
</boltAction>

<boltAction type="file" filePath="src/main.tsx">
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './index.css';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
</boltAction>

<boltAction type="file" filePath="src/App.tsx">
import React, { useState } from 'react';

interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

export default function App() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [input, setInput] = useState('');

  const addTodo = () => {
    if (input.trim()) {
      setTodos([...todos, { id: Date.now(), text: input, completed: false }]);
      setInput('');
    }
  };

  const toggleTodo = (id: number) => {
    setTodos(todos.map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    ));
  };

  const deleteTodo = (id: number) => {
    setTodos(todos.filter(todo => todo.id !== id));
  };

  return (
    <div className="min-h-screen bg-gray-100 py-8 px-4">
      <div className="max-w-md mx-auto bg-white rounded-lg shadow-md p-6">
        <h1 className="text-3xl font-bold text-center mb-6 text-gray-800">
          Todo App
        </h1>
        
        <div className="flex gap-2 mb-4">
          <input
            type="text"
            value={input}
            onChange={(e) => setInput(e.target.value)}
            onKeyPress={(e) => e.key === 'Enter' && addTodo()}
            placeholder="Add a new todo..."
            className="flex-1 px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
          <button
            onClick={addTodo}
            className="px-6 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition"
          >
            Add
          </button>
        </div>

        <ul className="space-y-2">
          {todos.map(todo => (
            <li
              key={todo.id}
              className="flex items-center gap-2 p-3 bg-gray-50 rounded-lg"
            >
              <input
                type="checkbox"
                checked={todo.completed}
                onChange={() => toggleTodo(todo.id)}
                className="w-5 h-5"
              />
              <span className={`flex-1 ${todo.completed ? 'line-through text-gray-500' : 'text-gray-800'}`}>
                {todo.text}
              </span>
              <button
                onClick={() => deleteTodo(todo.id)}
                className="px-3 py-1 bg-red-500 text-white rounded hover:bg-red-600 transition text-sm"
              >
                Delete
              </button>
            </li>
          ))}
        </ul>
      </div>
    </div>
  );
}
</boltAction>

<boltAction type="file" filePath="src/index.css">
@tailwind base;
@tailwind components;
@tailwind utilities;
</boltAction>

<boltAction type="file" filePath="tailwind.config.js">
export default {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {},
  },
  plugins: [],
};
</boltAction>

<boltAction type="file" filePath="postcss.config.js">
export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
</boltAction>

<boltAction type="file" filePath="tsconfig.json">
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
</boltAction>

<boltAction type="file" filePath="vite.config.ts">
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
});
</boltAction>

<boltAction type="shell">
npm install
</boltAction>

<boltAction type="shell">
npm run dev
</boltAction>
</boltArtifact>

Your React todo app is now running! You can add, complete, and delete todos.
```

#### 4. Parser Processing
```typescript
// StreamingMessageParser emits:
onArtifactOpen({ id: "react-todo-app", title: "React Todo App" })
onActionOpen({ type: "file", filePath: "package.json", content: {...} })
onActionClose()
onActionOpen({ type: "file", filePath: "index.html", content: "..." })
onActionClose()
// ... (all file actions)
onActionOpen({ type: "shell", content: "npm install" })
onActionClose()
onActionOpen({ type: "shell", content: "npm run dev" })
onActionClose()
```

#### 5. Action Execution
```typescript
// ActionRunner executes sequentially:
1. Write package.json → WebContainer
2. Write index.html → WebContainer
3. Write src/main.tsx → WebContainer
4. Write src/App.tsx → WebContainer
5. Write src/index.css → WebContainer
6. Write tailwind.config.js → WebContainer
7. Write postcss.config.js → WebContainer
8. Write tsconfig.json → WebContainer
9. Write vite.config.ts → WebContainer
10. Execute: npm install (status: running → complete)
11. Execute: npm run dev (status: running, server starts)
```

#### 6. Preview Ready
```typescript
// WebContainer emits:
'server-ready' event: { port: 5173, url: "https://..." }

// previewsStore updates:
addPreview(5173, "https://...")

// Preview component renders:
<iframe src="https://..." />

// User sees: Live React todo app running in browser!
```

#### 7. User Edits File
```typescript
// User changes button color in EditorPanel:
- className="px-6 py-2 bg-blue-500..."
+ className="px-6 py-2 bg-purple-500..."

// filesStore marks file as modified
// On next message, diff is generated:
diff --git a/src/App.tsx b/src/App.tsx
@@ -40,7 +40,7 @@
   <button
     onClick={addTodo}
-    className="px-6 py-2 bg-blue-500 text-white..."
+    className="px-6 py-2 bg-purple-500 text-white..."
   >

// Diff sent to AI with next user message for context
```

---

## Advanced Features

### Prompt Enhancement
- **Feature**: AI-powered prompt refinement
- **How it works**: Before sending, user clicks "enhance" icon
- **Implementation**: Separate API call to Claude to improve prompt clarity
- **Hook**: `usePromptEnhancer()` in `lib/hooks/`

### Chat History Persistence
- **Storage**: IndexedDB via `lib/persistence/db.ts`
- **Data**: Messages, artifacts, file states saved per chat session
- **Loading**: On mount, `useChatHistory()` restores previous chat
- **Sync**: Auto-saves on message updates

### File Diff Generation
- **Purpose**: Inform AI of user-made changes
- **Implementation**: `lib/utils/diff.ts` using `diff` package
- **Format**: Unified diff format (same as git diff)
- **Trigger**: Before each user message, diffs computed and prepended

### Multi-Preview Support
- **Scenario**: Multiple dev servers running (e.g., frontend + backend)
- **UI**: `PortDropdown` allows switching between preview URLs
- **Detection**: WebContainer emits `server-ready` for each port

### Terminal Integration
- **Library**: xterm.js
- **Purpose**: Display command output, allow user interaction
- **Implementation**: `Terminal.tsx` component
- **Features**: Scrollback, copy/paste, clickable links

### Syntax Highlighting
- **Library**: Shiki (VS Code's highlighter)
- **Usage**: Code blocks in messages, file content in editor
- **Languages**: 100+ supported (auto-detected from file extension)

### Theme Support
- **Themes**: Light & Dark modes
- **Storage**: `themeStore` (persisted in localStorage)
- **Application**: CSS variables, CodeMirror theme, xterm theme

---

## Security Considerations

### API Key Management
- **Storage**: Environment variable `ANTHROPIC_API_KEY`
- **Access**: Server-only (`lib/.server/llm/api-key.ts`)
- **Never exposed** to client-side code

### Sandbox Security
- **WebContainer**: Runs in isolated browser environment
- **No network access**: Can't make arbitrary external requests
- **File system**: Isolated from user's actual filesystem

### Content Sanitization
- **User input**: Escaped before rendering
- **AI output**: Markdown rendered with `rehype-sanitize`
- **XSS prevention**: React's built-in escaping

---

## Performance Optimizations

### Streaming
- **Benefit**: Users see responses immediately, not after completion
- **Implementation**: Web Streams API, incremental parsing
- **UX**: Reduces perceived latency

### Virtual Scrolling
- **Component**: `Messages.client.tsx`
- **Benefit**: Handles large message histories efficiently
- **Library**: Custom implementation with IntersectionObserver

### Code Splitting
- **Build**: Vite automatically splits routes and components
- **WebContainer**: Lazy-loaded only when needed
- **Result**: Faster initial page load

### Hot Module Reload (HMR)
- **Development**: Instant updates without full page reload
- **WebContainer**: State persisted across HMR via Vite metadata
- **Editor**: Document state preserved during updates

---

## Deployment

### Cloudflare Pages
- **Platform**: Edge deployment (low latency globally)
- **Build command**: `npm run build` (Remix + Vite)
- **Output directory**: `./build/client`
- **Functions**: Cloudflare Workers for API endpoints
- **Configuration**: `wrangler.toml`

### Environment Variables
```bash
ANTHROPIC_API_KEY=sk-ant-...
```

### Build Process
```bash
# Install dependencies
pnpm install

# Type check
pnpm typecheck

# Build for production
pnpm build

# Deploy to Cloudflare Pages
pnpm deploy
```

---

## Conclusion

Bolt.new achieves **real-time AI-driven full-stack development** through:

1. **Seamless AI Integration**: Claude 3.5 Sonnet generates complete, runnable code
2. **Browser-Based Runtime**: WebContainers enable Node.js apps in the browser
3. **Structured Communication**: XML artifacts provide clear separation of files and commands
4. **Reactive State Management**: Nanostores enable efficient UI updates
5. **Instant Feedback**: Streaming + live preview = immediate visual results

The architecture elegantly bridges the gap between natural language prompts and working applications, making full-stack development accessible to users of all skill levels.

---

## Additional Resources

- **GitHub Repository**: https://github.com/stackblitz/bolt.new
- **Contributing Guide**: [CONTRIBUTING.md](./CONTRIBUTING.md)
- **WebContainer Docs**: https://webcontainers.io/
- **Remix Docs**: https://remix.run/docs
- **Anthropic API**: https://docs.anthropic.com/
