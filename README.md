# claude-skills

A personal Claude Code plugin marketplace (`yaksh-skills`). One plugin, `core`, is installed globally. Every other plugin is installed per project, only where it is needed, so unrelated skills don't take up context.

## For Claude: choosing plugins for a project

When you start work in a project, check which plugins it needs:

1. Look at the project's files and dependencies (`package.json`, `pubspec.yaml`, `Cargo.toml`, `wrangler.toml`, `*.xcodeproj`, `src-tauri/`, and so on).
2. Match them against the **Signals** column below.
3. Check what is already installed with `claude plugin list`, and skip anything already there.
4. Tell the user which plugins match and why, then install them at project scope once they agree:

   ```
   claude plugin install <plugin>@yaksh-skills --scope project
   ```

   Project scope writes to the project's `.claude/settings.json`, so anyone who clones the project gets the same plugins.
5. Tell the user to restart Claude Code. New skills load only in a new session.

Never install a plugin at user scope except `core`. When no signal matches, install nothing.

## Plugins

| Plugin | Install when the project has | Skills |
|---|---|---|
| `core` | Installed globally; never install per project | Blog writing and SEO (`blog`, `blog-*`, `search-console`, `llms-visibility`), `ast-grep`, `worktree-create`, `worktree-merge`, `rules-check-drift`, `drive-screen`, `find-skills`, `mcp-builder`, `project-plugins` (reads this guide and suggests plugins) |
| `clerk` | `@clerk/*` dependency, `CLERK_*` env vars, or the user asks for auth with Clerk | 20 Clerk skills: setup, CLI, Backend API, orgs, billing, webhooks, testing, custom UI, and framework patterns for Next.js, React, React Router, TanStack, Vue, Nuxt, Astro, Expo, Swift, Android and Chrome extensions |
| `cloudflare` | `wrangler.toml` / `wrangler.jsonc`, `@cloudflare/*` dependency, Workers, Pages, D1, R2, KV or Durable Objects | `cloudflare`, `wrangler`, `durable-objects`, `workers-best-practices`, `agents-sdk`, `sandbox-sdk`, `cloudflare-email-service`, `web-perf` |
| `flutter` | `pubspec.yaml` with a Flutter SDK dependency | Widget and integration tests, widget previews, architecture, responsive layout, layout fixes, JSON serialization, `go_router` routing, localization, the `http` package |
| `mobile` | `*.xcodeproj`, `Package.swift`, SwiftUI code, Expo / React Native, or a Tauri app (`src-tauri/`) | `swiftui-pro` (SwiftUI review), `liquid-glass` (iOS/macOS 26 glass effects), `expo-motion` (Expo animation), `smooth-apple-motion` (Tauri v2 desktop and iOS field notes) |
| `web-design` | A web frontend where UI quality matters: landing pages, marketing sites, dashboards, animation or Three.js work | `impeccable`, `emil-design-eng`, `apple-design`, `review-animations`, `animation-vocabulary`, `threejs-shaders`, `threejs-postprocessing`, `smooth-apple-motion` (same skill as in `mobile`) |
| `media` | Image, video or GIF generation, Higgsfield, Remotion, or design assets | Six Higgsfield skills, `nano-banana`, `motion-graphics`, `video-downloader`, `canvas-design`, `slack-gif-creator`, `theme-factory` |
| `ai-dev` | An AI chat UI (`ai` / `@ai-sdk/*` with chat components), Composio, or LangChain / LangGraph agents | `ai-elements`, `composio`, `langsmith-fetch` |
| `rust` | `Cargo.toml` | `rust-skills` |
| `pinokio` | A Pinokio launcher (`pinokio.js`) or the user mentions Pinokio | `pinokio`, `gepeto` |
| `productivity` | Not tied to a codebase. Install when the user asks for one of these tasks. | `agent-reach` (web and social research), `second-brain-audit`, `second-brain-fix`, `tailored-resume-generator`, `domain-name-brainstormer`, `invoice-organizer`, `file-organizer`, `meeting-insights-analyzer`, `lead-research-assistant`, `competitive-ads-extractor`, `twitter-algorithm-optimizer`, `raffle-winner-picker`, `developer-growth-analysis` |

`smooth-apple-motion` is in both `mobile` and `web-design`, so a project with both installed loads it twice; that is harmless but costs a little context. A Tauri app usually needs `mobile` and `web-design`, and `rust` too if the Rust backend is substantial.

## Setup on a new machine

```
/plugin marketplace add ai-calypse/claude-skills
/plugin install core@yaksh-skills
```

The rest of the toolkit comes from other sources and is not in this repo:

- **gstack**: planning, review, shipping, QA and the headless browser. Installed in `~/.claude/skills/gstack` with its own setup script.
- **Skills synced from claude.ai**: docx, pdf, pptx, xlsx, docs, skill-creator and others. These arrive automatically with the claude.ai account.
- **Third-party plugin marketplaces**: vercel and frontend-design (`claude-plugins-official`), `neon-postgres`, `caveman`, `ponytail`, `typesafe`, and the animation-library plugins from `freshtechbro/claudedesignskills`.

## Updating

After pushing changes here, run `/plugin marketplace update yaksh-skills` to pull them into installed plugins.

## Layout

```
.claude-plugin/marketplace.json   lists every plugin
plugins/<plugin>/.claude-plugin/plugin.json
plugins/<plugin>/skills/<skill>/SKILL.md
```

To add a skill, put its folder under the plugin that fits, then update the skill count in `marketplace.json` and in that plugin's `plugin.json`.

## Credits

Most skills come from other authors' public repositories, including Clerk, Cloudflare, Flutter, Composio, Higgsfield, Anthropic's example skills, and Cole Medin (MIT, see `plugins/core/LICENSE-coleam00`). Their licenses still apply.
