# Blip roadmap — toward iMessage parity

Verified against fnix chat.db 2026-08-31: 27,118 tapbacks, 45,979 read receipts
on sent messages, 4,515 inline replies, 819 edits/unsends, 30 audio messages,
and a readable Contacts DB (3,694 records). Everything below Tier 4 needs no
changes to the Mac beyond claude-on-mac updates.

## Tier 1 — display parity (data already in chat.db, read-only)

- [ ] **Contact names + avatars** — AddressBook `.abcddb` → handle→name map via
  a new `imsg` query; initials circles first, contact photos later.
- [ ] **Read receipts (inbound)** — `message.date_read` → "Read 4:42 PM" under
  your last sent bubble. (Sending receipts stays impossible.)
- [ ] **Tapbacks on bubbles** — `associated_message_type` 2000–3007 → ❤️👍😂
  corner badges with who reacted.
- [ ] **Inline replies** — `thread_originator_guid` → quoted snippet above the
  bubble.
- [ ] **Edited / unsent markers** — `date_edited` / `date_retracted` → "edited"
  tag and "unsent a message" tombstone.
- [ ] **Attachment previews (stage 1)** — `message_attachment_join` → empty
  bubbles become "📷 photo" / "📄 PDF" chips; fixes blank inbound bubbles.
- [ ] **Send-effect labels** — `expressive_send_style_id` → "sent with Lasers".

## Tier 2 — attachments both directions (mechanisms proven 2026-08-31)

- [ ] **Inbound click-to-fetch** — scp over the existing mux; inline image view,
  `xdg-open` for PDF/video; HEIC→JPEG via `sips` on the Mac. OPEN DECISION:
  cache policy (`~/.cache/blip` size-capped vs tmpfs) — the no-content-on-disk
  invariant applies to state.json; fetched media needs its own rule.
- [ ] **Outbound files** — the Pictures-folder trick: scp → stage in
  `~/Pictures/` → AppleScript `send POSIX file` → cleanup. Verified delivered
  (PNG + PDF) 2026-08-31; from any other folder Messages fails with error 25.
  Lands as `imsg-send --file` in claude-on-mac; Blip compose gets attach +
  paste-image-from-clipboard.
- [ ] **Audio messages (inbound)** — fetch + `mpv` playback.

## Tier 3 — UX parity

- [ ] **Search** — panel UI (`/` key) over `imsg search`.
- [ ] **New-conversation composer** — contact picker from the AddressBook map.
- [ ] **Reply from the toast** — `notify-send` action buttons → inline reply.
- [ ] **Real-time push** — launchd watcher on chat.db pinging over the SSH mux
  (clipsync pattern) → sub-second delivery instead of the 6 s poll.
- [ ] **Link previews** — parse rich-link payload attachments for title/image.

## Tier 4 — the actual app

- [ ] **`blipd` daemon** — extract collector/thread/watcher behind a local
  socket; bar plugin becomes a thin client.
- [ ] **Standalone window** — QuickShell window reusing the existing QML,
  Messages.app layout (sidebar + conversation). Tauri v2 only if it ever needs
  to run off-Omarchy.

## Not possible (and why)

- **Outbound tapbacks / edits / typing indicators** — no public API; requires
  BlueBubbles-style code injection into Messages.app, which requires disabling
  SIP (System Integrity Protection) on the Mac. Rejected for fnix: daily
  driver, not worth the security downgrade.
- **Sending read receipts** — `open imessage://` does not flip `is_read`.
- **Group add/rename** — AppleScript cannot.
