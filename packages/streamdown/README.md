# Streamdown

A drop-in replacement for react-markdown, designed for AI-powered streaming.

[![npm version](https://img.shields.io/npm/v/streamdown)](https://www.npmjs.com/package/streamdown)

## Overview

Formatting Markdown is easy, but when you tokenize and stream it, new challenges arise. Streamdown is built specifically to handle the unique requirements of streaming Markdown content from AI models, providing seamless formatting even with incomplete or unterminated Markdown blocks.

Streamdown powers the [AI Elements Response](https://ai-sdk.dev/elements/components/response) component but can be installed as a standalone package for your own streaming needs.

## Features

- 🚀 **Drop-in replacement** for `react-markdown`
- 🔄 **Streaming-optimized** - Handles incomplete Markdown gracefully
- 🎨 **Unterminated block parsing** - Styles incomplete bold, italic, code, links, and headings
- 📊 **GitHub Flavored Markdown** - Tables, task lists, and strikethrough support
- 🔢 **Math rendering** - LaTeX equations via KaTeX
- 📈 **Mermaid diagrams** - Render Mermaid diagrams as code blocks with a button to render them
- 🎯 **Code syntax highlighting** - Beautiful code blocks with Shiki
- 🛡️ **Security-first** - Built with rehype-harden for safe rendering
- ⚡ **Performance optimized** - Memoized rendering for efficient updates

## Installation

```bash
npm i streamdown
```

Then, update your Tailwind `globals.css` to include the following.

```css
@source "../node_modules/streamdown/dist/index.js";
```

Make sure the path matches the location of the `node_modules` folder in your project. This will ensure that the Streamdown styles are applied to your project.

## Usage

### Basic Example

```tsx
import { Streamdown } from 'streamdown';

export default function Page() {
  const markdown = "# Hello World\n\nThis is **streaming** markdown!";

  return <Streamdown>{markdown}</Streamdown>;
}
```

### Mermaid Diagrams

Streamdown supports Mermaid diagrams using the `mermaid` language identifier:

```tsx
import { Streamdown, type MermaidConfig, type MermaidLoader } from 'streamdown';

const mermaidLoader: MermaidLoader = async () => (await import('mermaid')).default;

export default function Page() {
  const markdown = `
# Flowchart Example

\`\`\`mermaid
graph TD
    A[Start] --> B{Is it working?}
    B -->|Yes| C[Great!]
    B -->|No| D[Debug]
    D --> B
\`\`\`

# Sequence Diagram

\`\`\`mermaid
sequenceDiagram
    participant User
    participant API
    participant Database

    User->>API: Request data
    API->>Database: Query
    Database-->>API: Results
    API-->>User: Response
\`\`\`
  `;

  // Optional: Customize Mermaid theme and colors
  const mermaidConfig: MermaidConfig = {
    theme: 'dark',
    themeVariables: {
      primaryColor: '#ff0000',
      primaryTextColor: '#fff'
    }
  };

  return (
    <Streamdown mermaidConfig={mermaidConfig} mermaidLoader={mermaidLoader}>
      {markdown}
    </Streamdown>
  );
}
```

> **Note:** Mermaid support is optional. If you omit the `mermaidLoader` prop, Mermaid code fences will render as plain text. This keeps Streamdown safe to bundle in environments (like Chrome extensions) where `@mermaid-js/parser` cannot be resolved. Supply your own loader only when you need diagrams and can alias Mermaid’s dependencies appropriately.

### With AI SDK

```tsx
'use client';

import { useChat } from '@ai-sdk/react';
import { useState } from 'react';
import { Streamdown } from 'streamdown';

export default function Page() {
  const { messages, sendMessage, status } = useChat();
  const [input, setInput] = useState('');

  return (
    <>
      {messages.map(message => (
        <div key={message.id}>
          {message.parts.filter(part => part.type === 'text').map((part, index) => (
            <Streamdown isAnimating={status === 'streaming'} key={index}>{part.text}</Streamdown>
          ))}
        </div>
      ))}

      <form
        onSubmit={e => {
          e.preventDefault();
          if (input.trim()) {
            sendMessage({ text: input });
            setInput('');
          }
        }}
      >
        <input
          value={input}
          onChange={e => setInput(e.target.value)}
          disabled={status !== 'ready'}
          placeholder="Say something..."
        />
        <button type="submit" disabled={status !== 'ready'}>
          Submit
        </button>
      </form>
    </>
  );
}
```

### Customizing Plugins

When you need to override the default plugins (e.g., to configure security settings), you can import the default plugin configurations and selectively modify them:

```tsx
import { Streamdown, defaultRehypePlugins } from 'streamdown';
import { harden } from 'rehype-harden';

export default function Page() {
  const markdown = `
[Safe link](https://example.com)
[Unsafe link](https://malicious-site.com)
  `;

  return (
    <Streamdown
      rehypePlugins={[
        defaultRehypePlugins.raw,
        defaultRehypePlugins.katex,
        [
          harden,
          {
            defaultOrigin: 'https://example.com',
            allowedLinkPrefixes: ['https://example.com'],
          },
        ],
      ]}
    >
      {markdown}
    </Streamdown>
  );
}
```

The `defaultRehypePlugins` and `defaultRemarkPlugins` exports provide access to:

**defaultRehypePlugins:**
- `harden` - Security hardening with rehype-harden (configured with wildcard permissions by default)
- `raw` - HTML support
- `katex` - Math rendering with KaTeX

**defaultRemarkPlugins:**
- `gfm` - GitHub Flavored Markdown support
- `math` - Math syntax support

## Chrome Extension Compatibility

Streamdown is designed to work in restricted JavaScript environments like Chrome extensions. The library uses an optional dependency pattern for Mermaid diagrams to avoid bundling issues.

### The Problem

Chrome extensions have limitations that prevent standard bundling of certain dependencies:

- **Content Script Restrictions**: Content scripts run in a sandboxed context with limited module resolution
- **CSP Violations**: Some libraries (like Mermaid) use `Function()` calls that violate Content Security Policy
- **Bundler Resolution**: Dependencies like `@mermaid-js/parser` cannot be resolved in extension environments

### The Solution

Streamdown removes the hard dependency on `mermaid` and uses dependency injection instead:

```tsx
import { Streamdown, type MermaidLoader } from 'streamdown';

// Option 1: No mermaid support (safe for all environments)
<Streamdown>{markdown}</Streamdown>

// Option 2: Provide your own mermaid loader (when environment supports it)
const mermaidLoader: MermaidLoader = async () => (await import('mermaid')).default;
<Streamdown mermaidLoader={mermaidLoader}>{markdown}</Streamdown>
```

### Usage in Chrome Extensions

When bundling for Chrome extensions:

1. **Omit the `mermaidLoader` prop** - Mermaid code blocks render as plain text with a message
2. **No build errors** - Library bundles successfully without mermaid dependencies
3. **Full functionality** - All other features (code highlighting, math, tables) work normally

This pattern enables Streamdown to work across environments:
- ✓ Web applications (full mermaid support)
- ✓ Chrome extensions (mermaid optional)
- ✓ Server-side rendering (mermaid optional)
- ✓ Restricted JavaScript contexts (mermaid optional)

## Props

Streamdown accepts all the same props as react-markdown, plus additional streaming-specific options:

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `children` | `string` | - | The Markdown content to render |
| `parseIncompleteMarkdown` | `boolean` | `true` | Parse and style unterminated Markdown blocks |
| `className` | `string` | - | CSS class for the container |
| `components` | `object` | - | Custom component overrides |
| `rehypePlugins` | `array` | `[[harden, { allowedImagePrefixes: ["*"], allowedLinkPrefixes: ["*"], defaultOrigin: undefined }], rehypeRaw, [rehypeKatex, { errorColor: "var(--color-muted-foreground)" }]]` | Rehype plugins to use. Includes rehype-harden for security, rehype-raw for HTML support, and rehype-katex for math rendering by default |
| `remarkPlugins` | `array` | `[[remarkGfm, {}], [remarkMath, { singleDollarTextMath: false }]]` | Remark plugins to use. Includes GitHub Flavored Markdown and math support by default |
| `shikiTheme` | `[BundledTheme, BundledTheme]` | `['github-light', 'github-dark']` | The light and dark themes to use for code blocks |
| `mermaidConfig` | `MermaidConfig` | - | Custom configuration for Mermaid diagrams (theme, colors, etc.) |
| `mermaidLoader` | `MermaidLoader` | `undefined` | Lazily import Mermaid only when a diagram is rendered. When omitted, Mermaid fences render as plain text for maximum compatibility. |
| `controls` | `boolean \| { table?: boolean, code?: boolean, mermaid?: boolean }` | `true` | Control visibility of copy/download buttons |
| `isAnimating` | `boolean` | `false` | Whether the component is currently animating. This is used to disable the copy and download buttons when the component is animating. |

## Architecture

Streamdown is built as a monorepo with:

- **`packages/streamdown`** - The core React component library
- **`apps/website`** - Documentation and demo site

## Development

```bash
# Install dependencies
pnpm install

# Build the streamdown package
pnpm --filter streamdown build

# Run development server
pnpm dev

# Run tests
pnpm test

# Build packages
pnpm build
```

## Requirements

- Node.js >= 18
- React >= 19.1.1

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.
