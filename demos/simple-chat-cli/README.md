# Simple Chat CLI

A pedagogical, single-mode chat agent that introduces OpenAI's Conversations/Responses APIs in a hands-on way. It keeps one ongoing conversation, can call local tools, and can optionally search the web when it needs fresh information.

## What You'll Learn

This CLI demonstrates the essential building blocks of an OpenAI-powered agent:

1. Chat agent on the Conversations API (persistent state across turns)
2. Tool calling via the Responses API (function calls + web search)
3. File context injection using the `@` syntax
4. Token counting and session statistics
5. Local file utilities the model can call (create/move/edit files)

## Prerequisites

- Python 3.8+
- OpenAI API key (get one at https://platform.openai.com/api-keys)

## Installation

1. Navigate to this directory:
   ```bash
   cd demos/simple-chat-cli
   ```

2. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

3. Set up your API key:
   ```bash
   cp .env.example .env
   # Edit .env and add your OPENAI_API_KEY
   ```

## Usage

Run the chat application:

```bash
python chat.py
```

You will see a single prompt for the chat agent. All turns share context automatically.

```
[chat-agent]> Hello!
```

## Commands

All commands start with `/`. Press TAB after typing `/` to see available commands.

| Command | Description |
|---------|-------------|
| `/help` | Show help message |
| `/history` | View full conversation history |
| `/stats` | Show token usage and session statistics |
| `/new` | Start a new conversation |
| `/websearch` | Toggle web search on/off (enabled by default) |
| `/tools` | List the currently available tools |
| `/clear` | Clear the screen |
| `/exit` or `/quit` | Exit the application |

## Web Search

The model can automatically search the web when it needs current information.

- Enabled by default (toggle with `/websearch`)
- The model decides when to search
- Sources are displayed when used

Example:
```
[chat-agent]> What are the latest developments in AI this week?

🔍 Web Sources Used:
  1. Latest AI News - TechCrunch
     https://techcrunch.com/ai/...
  2. OpenAI Blog Updates
     https://openai.com/blog/...
```

## File References (@ syntax)

Include file contents in your prompts using `@`:

```
[chat-agent]> Summarize @README.md
[chat-agent]> Analyze @/absolute/path/to/file.py
[chat-agent]> What are the main points in @docs/intro.txt?
```

Features:
- Tab completion after `@` to complete file paths
- Relative paths (`@file.txt`, `@../dir/file.md`)
- Absolute paths (`@/usr/local/config.json`)
- Home directory expansion (`@~/Documents/notes.txt`)

## Tools the Agent Can Call

The agent can call local file utilities, and (when enabled) the built-in web search tool. Use `/tools` to see what is currently available.

- File utilities: `create_file`, `move_file`, `edit_file`
- Web search: built-in tool, toggle with `/websearch`

## Session Statistics

Use `/stats` to view token usage and session information (input tokens, output tokens, totals, and the current conversation ID).

## Example Sessions

### Example: Stateful Chat Agent

```
# Start chatting (stateful)
[chat-agent]> What is the capital of France?
Assistant: Paris is the capital of France.

[chat-agent]> What is its population?
Assistant: The population of Paris proper is about 2.1M; the metro area is much larger. (The model may search the web for current figures if needed.)
```

### Example: Using a File Reference

```
[chat-agent]> Please summarize @README.md
Assistant: <summary of this file>
```

---

This project is intentionally minimal to highlight the core patterns: a single, stateful chat agent that can read files you reference, call local tools, and optionally search the web when it needs fresh facts.