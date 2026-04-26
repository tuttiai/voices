# Tutti Voices — The Repertoire

The community registry of voices for the [Tutti](https://github.com/tuttiai/tutti) framework.

A **voice** is an npm package that gives an agent the ability to interact with an external system — a database, an API, the filesystem, a browser, anything reachable from Node. This repository is the index: machine-readable in [`voices.json`](./voices.json) and human-readable in the table below. Anyone can build a voice, publish it to npm, and open a PR to register it here.

---

## Official voices

12 official voices, 116+ tools.

| Voice | Package | Tools | Permissions | What it gives the agent |
|---|---|---|---|---|
| [discord](https://github.com/tuttiai/tutti/tree/main/voices/discord) | `@tuttiai/discord` | 11 | `network` | Read, post and moderate Discord messages |
| [filesystem](https://github.com/tuttiai/tutti/tree/main/voices/filesystem) | `@tuttiai/filesystem` | 7 | `filesystem` | Read, write, search, and manage local files |
| [github](https://github.com/tuttiai/tutti/tree/main/voices/github) | `@tuttiai/github` | 10 | `network` | Issues, PRs, repos, code search via the GitHub API |
| [mcp](https://github.com/tuttiai/tutti/tree/main/voices/mcp) | `@tuttiai/mcp` | dynamic | `network` | Wrap any MCP server — tools discovered at runtime |
| [playwright](https://github.com/tuttiai/tutti/tree/main/voices/playwright) | `@tuttiai/playwright` | 12 | `network`, `browser` | Browser automation: navigate, click, type, screenshot |
| [postgres](https://github.com/tuttiai/tutti/tree/main/voices/postgres) | `@tuttiai/postgres` | 8 | `network` | Query and inspect a PostgreSQL database (read-only by default) |
| [rag](https://github.com/tuttiai/tutti/tree/main/voices/rag) | `@tuttiai/rag` | 4 | `network` | Retrieval-augmented generation over your own docs |
| [sandbox](https://github.com/tuttiai/tutti/tree/main/voices/sandbox) | `@tuttiai/sandbox` | 4 | `shell` | Execute TypeScript, Python, or Bash in a sandboxed subprocess |
| [slack](https://github.com/tuttiai/tutti/tree/main/voices/slack) | `@tuttiai/slack` | 11 | `network` | Read, post and moderate Slack messages via a bot token |
| [stripe](https://github.com/tuttiai/tutti/tree/main/voices/stripe) | `@tuttiai/stripe` | 27 | `network` | Customers, payments, subscriptions, invoices, balance |
| [twitter](https://github.com/tuttiai/tutti/tree/main/voices/twitter) | `@tuttiai/twitter` | 9 | `network` | Post, read, search and manage Twitter/X content |
| [web](https://github.com/tuttiai/tutti/tree/main/voices/web) | `@tuttiai/web` | 3 | `network` | Search the web, fetch pages, read sitemaps |

Install any official voice with the shorthand name:

```bash
npx tutti-ai add filesystem
npx tutti-ai add stripe
```

Community voices use the full package name:

```bash
npx tutti-ai add @yourscope/your-voice
```

---

## Build your own voice

The full reference lives in the Tutti docs:

- [Building a Voice](https://tutti.ai/guides/building-a-voice) — quick walkthrough
- [Voice Authoring Guide](https://tutti.ai/contributing/voice-authoring) — comprehensive reference

The rest of this section is a self-contained crash course covering everything you need to publish a voice and get it merged into this registry.

### 1. What you're building

A voice is an npm package whose default export (or a named export) is a class that implements the `Voice` interface from [`@tuttiai/types`](https://www.npmjs.com/package/@tuttiai/types):

```ts
interface Voice {
  name: string;
  description?: string;
  tools: Tool[];
  required_permissions: Permission[];
  setup?(context: VoiceContext): Promise<void>;
  teardown?(): Promise<void>;
}
```

A `Tool` is a single callable function the LLM can invoke:

```ts
interface Tool<T = unknown> {
  name: string;                  // snake_case, verb-first (e.g. "list_issues")
  description: string;           // shown to the LLM — be explicit
  parameters: ZodType<T>;        // validated before execute() runs
  destructive?: boolean;         // true → automatically gated behind HITL approval
  execute(input: T, context: ToolContext): Promise<ToolResult>;
}
```

`Permission` is one of `"network"`, `"filesystem"`, `"shell"`, or `"browser"`. Be honest — only request what you actually need; users see these in their score file.

### 2. Project layout

```
my-voice/
├── src/
│   ├── index.ts            # Voice class + tool exports
│   └── tools/
│       ├── do-thing.ts     # one tool per file
│       └── read-thing.ts
├── tests/
│   ├── voice.test.ts
│   └── tools/
│       ├── do-thing.test.ts
│       └── read-thing.test.ts
├── package.json
├── tsconfig.json
├── tsup.config.ts
└── README.md
```

### 3. Write a tool

```ts
// src/tools/get-weather.ts
import { z } from "zod";
import type { Tool } from "@tuttiai/types";

const parameters = z.object({
  city: z.string().describe("City name (e.g. London, Tokyo)"),
  units: z.enum(["celsius", "fahrenheit"]).default("celsius")
    .describe("Temperature units"),
});

export const getWeatherTool: Tool<z.infer<typeof parameters>> = {
  name: "get_weather",
  description: "Get the current weather for a city",
  parameters,
  execute: async (input) => {
    try {
      const response = await fetch(
        `https://api.weather.example/v1?city=${encodeURIComponent(input.city)}&units=${input.units}`,
      );
      if (!response.ok) {
        return {
          content: `Weather API returned ${response.status}. Try a different city or check your API key.`,
          is_error: true,
        };
      }
      const data = await response.json();
      return { content: `Weather in ${input.city}: ${data.temp}° ${input.units}, ${data.condition}` };
    } catch (err) {
      return {
        content: `Failed to reach weather API: ${err instanceof Error ? err.message : String(err)}`,
        is_error: true,
      };
    }
  },
};
```

Rules of thumb that keep voices safe and predictable:

- **`execute()` never throws.** Catch every error path and return `{ content, is_error: true }` with a fix hint. The LLM uses the error message to decide what to do next; thrown exceptions abort the run.
- **Validate every input with Zod.** Never trust LLM-generated tool arguments — schemas catch malformed types and unexpected fields before they reach your code.
- **Mark destructive tools.** If the side effect is hard to undo (sending a message, deleting a row, charging a card), set `destructive: true`. Tutti's runtime auto-pauses these for operator approval (HITL) when enabled.
- **Use `.describe()` on every Zod field.** That text becomes the parameter doc the LLM sees.
- **No `process.env` in tool bodies.** Read secrets through the runtime's `SecretsManager` (or accept them as voice constructor options). This keeps them out of logs and event payloads.
- **No secrets in tool results.** The result string is sent back to the LLM and may end up in traces, logs, or screenshots.

### 4. Wrap tools in a Voice class

```ts
// src/index.ts
import type { Permission, Voice } from "@tuttiai/types";
import { getWeatherTool } from "./tools/get-weather.js";

export class WeatherVoice implements Voice {
  name = "weather";
  description = "Get weather data for any city";
  required_permissions: Permission[] = ["network"];
  tools = [getWeatherTool];
}

export { getWeatherTool };
```

If your voice needs to open a connection (DB pool, browser, websocket), do it lazily on first tool call or in `setup()`, and clean up in `teardown()`. Voices are constructed eagerly during score loading, so synchronous constructors must not throw on missing config — return an `is_error` from each tool instead. This is the "lazy + fail-soft" pattern every official voice uses.

### 5. Configure the package

```json
{
  "name": "@yourscope/my-voice",
  "version": "0.1.0",
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "files": ["dist"],
  "dependencies": {
    "@tuttiai/types": "*",
    "zod": "3.25.76"
  },
  "scripts": {
    "build": "tsup",
    "test": "vitest run"
  }
}
```

```ts
// tsup.config.ts
import { defineConfig } from "tsup";

export default defineConfig({
  entry: ["src/index.ts"],
  format: ["esm"],
  dts: true,
  clean: true,
});
```

### 6. Test it

Three layers — all three should exist before publishing:

```ts
// Unit test: each tool in isolation
import { describe, it, expect } from "vitest";
import { getWeatherTool } from "../src/tools/get-weather.js";

const ctx = { session_id: "test", agent_name: "test" };

describe("getWeatherTool", () => {
  it("returns formatted weather", async () => {
    const result = await getWeatherTool.execute({ city: "London", units: "celsius" }, ctx);
    expect(result.content).toContain("London");
    expect(result.is_error).toBeUndefined();
  });

  it("returns is_error on API failure", async () => {
    // mock fetch → 500
    const result = await getWeatherTool.execute({ city: "Atlantis", units: "celsius" }, ctx);
    expect(result.is_error).toBe(true);
  });
});
```

```ts
// Contract test: voice implements the interface
import { WeatherVoice } from "../src/index.js";

describe("WeatherVoice", () => {
  it("declares the right shape", () => {
    const voice = new WeatherVoice();
    expect(voice.name).toBe("weather");
    expect(voice.required_permissions).toEqual(["network"]);
    expect(voice.tools.length).toBeGreaterThan(0);
  });
});
```

```ts
// Integration test: end-to-end through AgentRunner with a mock provider
import { AgentRunner, EventBus, InMemorySessionStore } from "@tuttiai/core";
import { createMockProvider, toolUseResponse, textResponse }
  from "@tuttiai/core/tests/helpers/mock-provider";

const provider = createMockProvider([
  toolUseResponse("get_weather", { city: "London", units: "celsius" }),
  textResponse("It's 18°C in London."),
]);

const runner = new AgentRunner(provider, new EventBus(), new InMemorySessionStore());
const result = await runner.run(
  {
    name: "test",
    system_prompt: "Test",
    voices: [new WeatherVoice()],
    permissions: ["network"],
  },
  "What's the weather in London?",
);
expect(result.output).toBe("It's 18°C in London.");
```

Coverage targets matching the official voices: **80% lines, 80% functions, 70% branches.** No real API calls in tests — mock everything.

### 7. Publish to npm

Pre-flight checklist:

- [ ] All tools have descriptive names (`snake_case`, verb-first) and `description` fields
- [ ] All Zod parameters use `.describe()` annotations
- [ ] `required_permissions` is accurate and minimal
- [ ] Destructive tools are flagged with `destructive: true`
- [ ] `execute()` never throws — every error path returns `{ content, is_error: true }`
- [ ] Tests pass with ≥80% coverage
- [ ] No secrets, `.env` files, or credentials in the published package
- [ ] `npm audit --audit-level=high` is clean
- [ ] README explains what the voice does, env vars it reads, and a usage example

Then:

```bash
npm publish --access public
```

### 8. Register your voice here

Open a PR adding an entry to [`voices.json`](./voices.json) under `"community"`:

```jsonc
{
  "name": "weather",                                    // shorthand (alphanumeric + dashes)
  "package": "@yourscope/weather-voice",                // exact npm package name
  "description": "Get current weather for any city",    // one line, no marketing
  "repo": "https://github.com/yourscope/weather-voice", // public source
  "version": "0.1.0",                                   // matches the published version
  "author": "yourscope",                                // npm scope or GitHub handle
  "permissions": ["network"],                           // required_permissions array
  "tools_count": 1,                                     // integer, or "dynamic" for runtime-discovered
  "tags": ["weather", "forecast", "api"]                // 3-6 lowercase tags
}
```

PR checklist:

- [ ] Package is published and installable from npm
- [ ] Repo is public, with a README and a license
- [ ] Entry validates against the `voices.json` shape (use any existing entry as a template)
- [ ] Entries stay sorted alphabetically by `name` within their section

A maintainer reviews the package surface (no malware, no obvious security holes, voice interface implemented correctly) and merges. Once merged, your voice is installable via `npx tutti-ai add @yourscope/weather-voice` and discoverable through `tutti-ai voices search`.

---

## Reporting a malicious or broken voice

Open an issue with the `voice:report` label, or email security@tutti.ai for anything sensitive. Listed voices that go unmaintained for >12 months may be moved to an `archived` section.
