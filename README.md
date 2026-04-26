# Tutti Voices — The Repertoire

The community registry of voices for the Tutti framework.
Anyone can compose a voice and submit it here.

## Official voices

| Voice | Package | Description |
|---|---|---|
| filesystem | `@tuttiai/filesystem` | Read and write files on the local filesystem |
| github | `@tuttiai/github` | Interact with GitHub repositories, issues, and pull requests |
| playwright | `@tuttiai/playwright` | Control a browser like a human for QA and automation |
| slack | `@tuttiai/slack` | Read, post and moderate Slack messages via a bot token |
| postgres | `@tuttiai/postgres` | Query and inspect a PostgreSQL database (read-only by default) |

## Submit your voice

1. Build it with `tutti-ai create voice my-voice`
2. Publish to npm as `@tuttiai/voice-my-voice`
3. Open a PR adding it to `voices.json`
