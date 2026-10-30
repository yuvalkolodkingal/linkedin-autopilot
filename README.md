# linkedin-weekly-poster

A Claude skill that sets up automatic weekly LinkedIn posts at random times inside a window you choose. It runs as two layers of scheduled tasks, so the timing is random and the model never picks the numbers.

Say **"set up the LinkedIn weekly poster"** in Claude, and it will ask for your topics and schedule, then install everything.

## How it works

```mermaid
flowchart TD
    C["Cron: planner<br/>first allowed day, window start minus 1 hour"] --> P["Planner: Python picks a random<br/>allowed day and minute"]
    P -->|"create_trigger, run_once_at in UTC"| O["One-shot poster task"]
    O --> T["Pick least-recently-used topic (Python)"]
    T --> W["Write post: write-like-me, then humanizer"]
    W -->|"mode = approve"| A["Send draft to you, wait for 'approve'"]
    W -->|"mode = auto"| X["Publish via Composio<br/>LINKEDIN_CREATE_LINKED_IN_POST"]
    A --> X
    X --> L["Append to post log in memory, reply with post ID"]
```

- **Planner.** A recurring cron task. Cron cannot pick a random time, so the planner runs a short Python snippet in bash to choose a random allowed weekday in the coming week and a random minute inside the window. It converts that to UTC and creates a one-shot poster task for that moment.
- **Poster.** The one-shot task. It reads your config from memory, picks the topic that was used longest ago (never last week's), writes the post in your voice, publishes it, logs it, and reports the post ID.

## Requirements

- Claude with scheduled-task tools: `create_trigger`, `list_triggers`, `update_trigger`, `delete_trigger`, `fire_trigger`.
- [Composio](https://composio.dev) with a connected LinkedIn account. In claude.ai, add a custom connector named `composio` with URL `https://connect.composio.dev/mcp`, then connect LinkedIn inside it.
- Memory enabled. The config and post log live in `/areas/linkedin-poster.md`.
- Optional: the `write-like-me` and `humanizer` skills. Without them, posts use a plain conversational voice.

## Install

1. Add this skill to Claude. Copy this folder to your skills directory, or install it from the repository.
2. Set up Composio and connect LinkedIn, as described above.
3. Optional: set up `write-like-me` so posts match your voice. The skill offers this if it finds no profile.
4. In a Claude conversation, say: **"set up the LinkedIn weekly poster"**.
5. Answer the questions. Claude asks only for what it lacks:
   - **topics**: 3 to 10 lines, one topic per line. Required.
   - **language**: the language you write in, or `alternate` or `random`.
   - **allowed_days**: Sun–Thu (default), Mon–Fri, or all 7.
   - **window**: default 08:00–12:00.
   - **timezone**: taken from your session.
   - **mode**: `auto` posts on its own. `approve` sends you the draft first.
   - **visibility**: `PUBLIC` by default.
6. Check the permission mode of the two scheduled tasks. Unattended posting needs **Automatically approve**. If a task is set to **Ask**, switch it in the task settings.

## Changing or pausing

- **Change topics, language, style, or mode:** edit the Config section of `/areas/linkedin-poster.md`. The next run uses the new values.
- **Change days, window, or timezone:** edit the memory file, then ask Claude to update the planner's schedule. Claude will also clear any post already pending for this week.
- **Pause:** ask Claude to disable the planner and delete the pending post.
- **Resume:** ask Claude to re-enable the planner.

## Limits

- LinkedIn posts must be 3,000 characters or fewer. The poster aims for 600 to 1,300 and never exceeds 2,900.
- The LinkedIn API cannot read your profile text, so the voice comes only from `write-like-me`.
- `LINKEDIN_GET_MY_INFO` is rate-limited. Claude resolves your author URN once, during setup.

## License

MIT. See [LICENSE](LICENSE).
