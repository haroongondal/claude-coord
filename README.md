# claude-coord

A Claude Code plugin that lets several Claude Code sessions on one machine take turns on the same file, folder, app or device instead of colliding.

Claude Code sessions can already message each other (`SendMessage`), but nothing records who is using what. So one session can't tell that another is halfway through editing a file or driving the test phone. `coord` adds that shared record: one queue per resource, kept in `~/.claude/coord/queues.json`. Sessions still talk to each other with `SendMessage`.

## What it does

- **Files are claimed automatically.** A `PreToolUse` hook claims each file a session edits. If another session already has that file, or a folder above it, the edit is refused and the message says who has it and how to queue. A `Stop` hook frees the session's files when its turn ends.
- **Devices, apps and folders are claimed explicitly.** Run `coord claim device:<id>` before driving a phone, emulator or simulator, or `coord claim <folder>` before work that must not interleave with another session.
- **Queues.** A session that has to wait runs `coord wait <key>` in the background, which wakes it as soon as the resource is its turn. Meanwhile it messages the holder and works on something else. A third session queues behind the second.
- **Lending.** A holder that is only waiting on a device (an install, a long run) can `coord lend` it to the first session in the queue, with a note saying what the borrower must not touch. When the holder needs it again, `coord back` asks for it back, and the borrower runs `coord yield` once it has stopped.
- **Closed sessions drop out.** If a session's process is gone, its places in every queue are removed.

A `SessionStart` hook injects the rules ([PROTOCOL.md](plugins/coord/PROTOCOL.md)) into every session, so no CLAUDE.md edit is needed.

## Install

```bash
claude plugin marketplace add haroongondal/claude-coord
claude plugin install coord@claude-coord --scope user
```

Restart open sessions to load it. To make a project ask for it, add this to the project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "claude-coord": { "source": { "source": "github", "repo": "haroongondal/claude-coord" } }
  },
  "enabledPlugins": { "coord@claude-coord": true }
}
```

## Commands

| Command | What it does |
| --- | --- |
| `coord claim KEY [note]` | Take KEY, or join its queue. Exit 0 means it is yours; exit 3 means you are queued. |
| `coord wait KEY` | Block until KEY is yours. Run it in the background. |
| `coord release KEY` | Leave KEY's queue and say who has it next. |
| `coord lend KEY [note]` | Let the next session borrow KEY while you wait on it. |
| `coord back KEY` | End the loan, then `coord wait KEY` until the borrower gives it back. |
| `coord yield KEY` | Give a borrowed KEY back and rejoin the end of the queue. |
| `coord status [KEY]` | Show the queues. |

KEY is a path, `device:<id>` or `app:<package>`.

## Limits

- macOS and Linux only: it uses `fcntl` file locks.
- It works out which sessions are alive and what they are called from Claude Code's internal `~/.claude/sessions/<pid>.json` files. If a Claude Code update changes that format, coord stops refusing anything instead of blocking every edit.
- Nothing checks device commands such as `adb` or `xcrun simctl`. Device claims work only if sessions follow the rules.
- Needs `python3` on `PATH`.

## Tests

```bash
python3 plugins/coord/bin/coord_test.py
```

## License

MIT
