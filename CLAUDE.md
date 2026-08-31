# Blip — agent notes

Blip is an Omarchy (QuickShell/QML) bar plugin that puts iMessage on Linux by
treating a Mac as the gateway. Read this before touching anything.

## Shape

```
(Mac side)    imsg, imsg-send      from github.com/nixfred/claude-on-mac ≥ 1.4.0 — sqlite read of chat.db;
                                    AppleScript send; `imsg --json groups`; Recently Deleted hidden.
(Linux side)  ~/bin/imsg, imsg-send claude-on-mac's bin/remote/ ssh shims (CLAUDE_ON_MAC_TARGET), or any
                                    equivalent that execs the Mac tool over ssh. Blip only calls ~/bin/imsg*.
collector.ts                        poll → {threads, unread, toast}. Pure functions + one spawn.
thread.ts                           one conversation → decorated bubbles. Pure + one spawn.
BarWidget.qml                       the single poller, badge, toasts, IPC.
Panel.qml                           list view + conversation view + compose. Renders only.
manifest.json                       plugin id nixfred.blip
```

Logic lives in TypeScript where it can be unit-tested (`bun test`). QML renders
what it is handed. Keep it that way.

## Invariants — do not break

- **Never send to a handle for a group.** A group thread's `handle` is whichever
  member spoke last. `isSendable()` tests the *chat* id; groups send
  `--chat-id <full guid>` (`any;+;<32hex>`), DMs send `--to <chat>`.
- **Two read marks.** `watermark` = what the collector has seen (drives toasts).
  `readMark` / `readMarks[chat]` = what the user has looked at (drives the badge
  and the blue dots). Collapsing them makes the badge flash and reset.
- **Unread is ledger-backed.** `unreadCounts` and `unreadOldest` persist per-chat
  metadata without message bodies. Catch-up fetches cover new arrivals and the
  oldest outstanding unread so deletions are reconciled; never derive the total
  badge solely from the preview window.
- **Dedupe the self-thread before counting.** A message you send yourself lands
  twice (`from_me` true and false, same ts+text). `dedupeSelfEcho()` runs before
  `buildThreads()` and before `decorate()`.
- **`PanelKeyCatcher` eats keys before focused children.** Any editor that
  should receive typing must be covered by its `blocked:` binding.
- **Pass `--` before message text** to both `imsg-send` and `notify-send`.
- **No message content on disk.** `~/.local/state/blip/state.json` holds
  timestamps, counts, opaque SHA-256 toast keys, self-chat ids, and group
  metadata. It is atomic and `0600`; no message bodies are allowed.
- **`chat:null` exists.** Use `chatKey()`; never `String(m.chat)`.
- **Group ids come in two shapes**: 32 hex, or `chat<digits>`. `isGroupChat()` is
  "not a phone/email" — never a positive regex on one shape.
- **Deleted messages stay in chat.db for 30 days** (`chat_recoverable_message_join`).
  claude-on-mac's `imsg` hides them (1.4.0+); Blip assumes that.

## Working on it

```
bun test                                   # 90+ tests, ~40 ms
bun collector.ts --deep | jq .unread       # live against the Mac
bun thread.ts <chat-id> 40 | jq .bubbles   # one conversation
cp *.qml *.ts manifest.json ~/.config/omarchy/plugins/nixfred.blip/
omarchy-restart-shell                      # new QML files need a restart; edits hot-reload
qs -p /usr/share/omarchy/shell ipc call nixfred.blip status
qs -p /usr/share/omarchy/shell ipc call nixfred.blip goto 15550100001   # bare digits: qs rejects a leading "+"
```

IPC function names must not collide with `qs ipc` subcommands (`show` did).

## Verifying UI changes

Screenshot it: `grim -g "1300,40 620x860" out.png` with the panel open, then
look at the image. QML has no unit tests; every layout bug so far was found
that way, not by reading code.

Never inject keystrokes (`wtype`) unless the panel is confirmed open — they go
to whatever has focus otherwise.

## Things that are not possible

- Read receipts. `open imessage://<handle>` on the Mac does not flip `is_read`.
- Tapbacks, edits, threads, attachments out. AppleScript cannot.
