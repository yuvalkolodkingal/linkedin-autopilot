---
name: linkedin-weekly-poster
description: Set up automatic, scheduled weekly LinkedIn posts with randomized timing. Use when the user says "set up the LinkedIn weekly poster", or asks for automatic, scheduled, or weekly LinkedIn posting, a LinkedIn posting bot, or LinkedIn posts at random times. Installs a planner scheduled task and a one-shot poster task, publishes through Composio, and keeps a post log in memory.
---

# LinkedIn weekly poster

Installs two layers of scheduled tasks:

- **Planner**: a recurring cron task. It fires one hour before the posting window opens, on the first allowed weekday, in the user's timezone. It picks a random allowed day and a random minute in the window, converts that to UTC, and creates a one-shot poster task at that time.
- **Poster**: the one-shot task. It picks the least-recently-used topic, writes the post in the user's voice, publishes it through Composio, appends to the post log in memory, and replies with the text and post ID.

Cron cannot randomize, so the planner does that. The model never chooses the numbers. Bash does.

## Prerequisites

- Scheduled-task tools: `create_trigger`, `list_triggers`, `update_trigger`, `delete_trigger`, `fire_trigger`.
- Composio with an ACTIVE LinkedIn connection.
- Memory enabled. The config lives at `/areas/linkedin-poster.md`.

## Setup

Follow these steps in order. Do not create any scheduled task until step 1 passes.

### 1. Connection check

1. Call `COMPOSIO_SEARCH_TOOLS` with the use case "create a text post on LinkedIn as the authenticated user".
2. If no Composio tools are available at all, stop. Tell the user to add a custom connector in claude.ai named `composio` with URL `https://connect.composio.dev/mcp`, then connect LinkedIn inside it. Come back after that.
3. If there is no ACTIVE LinkedIn connection, call `COMPOSIO_MANAGE_CONNECTIONS` with `{"name": "linkedin", "action": "add"}`. Show the auth link as a markdown link and wait for the user. Then re-check with `COMPOSIO_MANAGE_CONNECTIONS` using action `list`.
4. Resolve the author URN once with `LINKEDIN_GET_MY_INFO`. The result is `urn:li:person:<id>`. Bake it into the prompts and the memory file. Do not call this tool again, because it throttles with HTTP 429.

### 2. Style skills

Check whether the `write-like-me` and `humanizer` skills are available (`ListSkills` or `SearchSkills`).

- If `write-like-me` is present but has no profile yet, offer its setup first, then continue.
- Record `style` as the subset that exists: `write-like-me, humanizer`, `write-like-me`, `humanizer`, or `plain`.
- If neither exists, write in a plain conversational voice.

### 3. Intake: ask only for what is missing

| Field | Required | Default |
|---|---|---|
| `topics` | yes | none. 3–10 lines, kept verbatim. One topic per line. |
| `language` | no | the language the user is writing in now. Or `alternate` or `random`. |
| `allowed_days` | no | Sun–Thu. Or Mon–Fri, or all 7. |
| `window` | no | 08:00–12:00 |
| `tz` | no | the IANA timezone from the session |
| `mode` | no | `auto`. Or `approve`. |
| `visibility` | no | `PUBLIC` |

### 4. Write the memory file

Write `/areas/linkedin-poster.md` using the template in the [Memory file](#memory-file) section. Fill in every config line now. The planner and poster read this file each time they run.

### 5. Create the planner

Compute the cron expression:

- Planner time = window start minus one hour, in the user's timezone.
- Day of week = the cron weekday of the first allowed day. Cron uses Sun=0 through Sat=6.
- Format: `CRON_TZ=<tz> <minute> <hour> * * <dow>`.

| Example | Cron expression |
|---|---|
| Sun–Thu, 08–12, Israel | `CRON_TZ=Asia/Jerusalem 0 7 * * 0` |
| Mon–Fri, 08–12, New York | `CRON_TZ=America/New_York 0 7 * * 1` |

If the window starts before 01:00, the planner's one-hour lead falls on the previous calendar day. Shift the day-of-week back by one, and tell the user you did.

Create the planner with `create_trigger`:

- `name`: `LinkedIn weekly poster: planner`
- `cron_expression`: from the table above
- `prompt`: the planner prompt from the [Planner prompt](#planner-prompt) section. Replace `<<POSTER PROMPT>>` with the poster prompt from the [Poster prompt](#poster-prompt) section, with `<AUTHOR_URN>` filled in.
- `create_new_session_on_fire`: `true`
- `connectors`: `["composio"]`. Pass this only because the user asked for LinkedIn posting. If the result warns that no connectors are stored, tell the user plainly and pass on the remedy it names.
- `initiation`: `human_request`

Write the returned trigger ID into the memory file as `planner_trigger`.

### 6. Confirm to the user

Report all of the following:

- The planner's day and time, and the window.
- Whether the first post lands this coming week. To post now instead, use `fire_trigger` on the planner. Firing it twice queues two posts, but the planner's guard stops a second pending post.
- The permission mode of the planner and poster tasks. Unattended posting needs **Automatically approve**. If a task got **Ask**, tell the user to switch it in the task settings.

## Planner prompt

````
You are the planner for the LinkedIn weekly poster. Run these steps exactly. Do not improvise.

1. Read `/areas/linkedin-poster.md` from memory. Take `tz`, `allowed_days`, `window`, `visibility`, `mode`, `author_urn`, and `planner_trigger` from `## Config`.

2. Guard against double-firing. Call `list_triggers` with enabled set to true. If a pending one-shot whose name starts with "LinkedIn post: " has a `run_once_at` in the future, stop and reply: "A post is already scheduled for <time>. Nothing created."

3. Run this in bash after replacing the placeholders with config values. Map `allowed_days` to Python weekdays as follows: Sun–Thu is `6, 0, 1, 2, 3`, Mon–Fri is `0, 1, 2, 3, 4`, all 7 is `0, 1, 2, 3, 4, 5, 6`. Do not change the logic.

```python
import random, datetime, zoneinfo

tz = zoneinfo.ZoneInfo("<TIMEZONE>")
allowed = {<ALLOWED_PY_WEEKDAYS>}                # Mon=0 .. Sun=6
start_min = <WINDOW_START_HOUR> * 60 + <WINDOW_START_MIN>
end_min = <WINDOW_END_HOUR> * 60 + <WINDOW_END_MIN>

now = datetime.datetime.now(tz)
days = [now.date() + datetime.timedelta(days=d) for d in range(7)]
days = [d for d in days if d.weekday() in allowed]

slot = None
for _ in range(50):
    day = random.choice(days)
    minute = random.randrange(start_min, end_min)
    candidate = datetime.datetime(day.year, day.month, day.day, minute // 60, minute % 60, tzinfo=tz)
    if candidate > now + datetime.timedelta(minutes=5):
        slot = candidate
        break

if slot is None:
    raise SystemExit("no slot found: check allowed_days and window")

print(slot.astimezone(datetime.timezone.utc).strftime("%Y-%m-%dT%H:%M:00Z"))
print(slot.strftime("%A %d/%m %H:%M"))
```

4. The first output line is the UTC time in RFC3339 format. The second line is the local time. If the script fails, stop and report the error.

5. Fill `<AUTHOR_URN>` in the poster prompt if it is still a placeholder. Then create the one-shot poster task with `create_trigger`:
   - `name`: "LinkedIn post: <local time from step 3>"
   - `run_once_at`: the UTC time from step 3
   - `create_new_session_on_fire`: true
   - `connectors`: ["composio"]
   - `initiation`: "human_schedule"
   - `prompt`: the poster prompt, copied verbatim

6. In the memory file's `## Config`, set `next_post` to the local time.

7. Reply with the local time and the new trigger ID.

<<POSTER PROMPT>>
````

## Poster prompt

````
You are the poster for the LinkedIn weekly poster. The planner scheduled you to publish one post. Publish it, log it, and report back. Author URN: <AUTHOR_URN>.

1. Read `/areas/linkedin-poster.md` from memory. Use `## Config` (topics, language, mode, visibility, style) and `## Post log`.

2. Pick the topic in bash. Copy the topics and the post-log lines verbatim from memory into the script. The script makes the choice. Do not pick a topic yourself.

```python
import random
topics = [<TOPICS_PY_LIST>]
log = [l for l in """<POST_LOG_LINES>""".splitlines() if l.strip()]

def last_used(t):
    hits = [i for i, l in enumerate(log) if f" — {t} — " in l]
    return hits[-1] if hits else -1

last = next((t for t in topics if log and f" — {t} — " in log[-1]), None)
pool = [t for t in topics if t != last] or topics
oldest = min(last_used(t) for t in pool)
print(random.choice([t for t in pool if last_used(t) == oldest]))
```

Use the printed topic. Do not repeat last week's topic.

3. Choose the language. If `language` is `en` or `he`, use it. If it is `alternate`, use the opposite of the language of the last log entry. A log entry is Hebrew if it contains Hebrew letters. If it is `random`, run `python3 -c "import random; print(random.choice(['en', 'he']))"`.

4. Facts. If the post states any fact, number, client name, or quote, verify it with web search, or write around it. Never invent facts, numbers, clients, or quotes.

5. Write the post.
   - Voice: if `style` includes `write-like-me`, follow its profile. If `style` includes `humanizer`, run the draft through it last. If `style` is `plain`, write in a plain conversational voice.
   - Hook: one concrete line. Do not open with "I've been thinking", and do not ask a question nobody is there to answer.
   - One idea per post. Short paragraphs. No bullet walls. No strings of emoji.
   - Length: 600 to 1,300 characters. Hard maximum 2,900. LinkedIn rejects anything over 3,000. Check the length in bash before continuing.
   - Hashtags: at most 3, at the end. Or none.
   - Hebrew: natural spoken Hebrew. Avoid bureaucratic nominalisations.

6. If `mode` is `approve`, send the draft to the user with SendUserMessage and stop. Publish only after the user replies "approve" in this session. If they ask for changes, revise the draft and send it again.

7. Publish with Composio. Fetch the input schema of `LINKEDIN_CREATE_LINKED_IN_POST` with `COMPOSIO_GET_TOOL_SCHEMAS` first. Then create the post with author `<AUTHOR_URN>`, the post text, and the `visibility` from config. If the call fails, retry once. If it fails again, stop and reply with the full draft and the exact error.

8. Log it. Append one line to `## Post log` in memory, in this format: `- <YYYY-MM-DD> — <topic> — <first 60 characters of the post, newlines replaced by spaces>`. Use the date in the config timezone.

9. Reply with the full post text and the post ID returned by the create call. Do not call `GET_POST_CONTENT` to verify the post. It can return 403 after a successful create.
````

## Memory file

Template for `/areas/linkedin-poster.md`:

```markdown
---
name: linkedin-poster
description: Config and post log for the LinkedIn weekly poster skill
sources: [chat]
---

## Config
- topics:
  - <topic 1>
  - <topic 2>
  - <topic 3>
- language: <en | he | alternate | random>
- allowed_days: <Sun,Mon,Tue,Wed,Thu>
- window: <08:00-12:00>
- tz: <IANA zone>
- mode: <auto | approve>
- visibility: <PUBLIC>
- style: <write-like-me, humanizer | write-like-me | humanizer | plain>
- author_urn: <urn:li:person:...>
- planner_trigger: <trig_...>
- next_post: <local time, set by the planner>

## Post log
- <YYYY-MM-DD> — <topic> — <first 60 characters>
```

Each post adds one line to `## Post log`. Keep the lines in chronological order, because the topic picker uses the order.

## Maintenance

- **Change topics, language, style, visibility, or mode:** edit `## Config` in the memory file. The next run reads it.
- **Change `allowed_days`, `window`, or `tz`:** edit the memory file, then call `update_trigger` on the planner with the new `cron_expression`. If a post is already pending for this week, delete it with `delete_trigger` so the new schedule applies.
- **Change the author URN or the poster prompt:** update the planner with `update_trigger` and a new `prompt`.
- **Pause:** call `update_trigger` on the planner with `enabled: false`. Then call `list_triggers`, find the pending one-shot whose name starts with "LinkedIn post: ", and delete it with `delete_trigger`.
- **Resume:** call `update_trigger` on the planner with `enabled: true`.

## Post rules

- 600–1,300 characters. Hard maximum 2,900.
- Concrete hook line. No "I've been thinking". No question to nobody.
- One idea per post. Short paragraphs. No bullet walls. No emoji strings.
- At most 3 hashtags at the end, or none.
- Hebrew: natural spoken Hebrew. No bureaucratic nominalisations.
- No invented facts, numbers, clients, or quotes. Verify with web search or write around it.
- If publishing fails after one retry, stop and reply with the draft and the exact error.

## Pitfalls

- `LINKEDIN_GET_MY_INFO` throttles with HTTP 429. Resolve the author URN once, during setup.
- A post over 3,000 characters is rejected with HTTP 400. Keep it under 2,900.
- `GET_POST_CONTENT` can return 403 after a successful create. Do not use it to verify a post.
- Firing the planner twice queues two posts. The planner's guard prevents a second pending post, but do not fire it by hand while one is pending.
- The LinkedIn API cannot read the user's own profile text. The voice comes only from `write-like-me`.
- Unattended posting needs the **Automatically approve** permission mode on both tasks.
