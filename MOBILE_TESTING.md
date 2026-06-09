# Mobile QA — Manual Testing Checklist

**Applies to:** Signalix-mobile v0.16.0 foundation MVP.
**Surface under test:** Android emulator running Expo Go SDK 54, talking to a locally-running Signalix backend.

This is a manual checklist because v0.16.0 has no automated test suite yet. Run every section before claiming an issue is fixed or a release is ready.

---

## 0. Prerequisites

Before opening the emulator:

- [ ] Docker stack up: `cd Signalix-infra && ./scripts/up.sh` — confirm `http://localhost:4000/api/v1/health` returns OK and `ws://localhost:5000` accepts connections (browser devtools is fine).
- [ ] Contracts dist built: `cd Signalix-contracts && npm install && npm run build` — confirm `dist/protocol/index.js` exists.
- [ ] Mobile deps installed: `cd Signalix-mobile && npm install`.
- [ ] Emulator picks up host backend: `app.json` → `extra.apiUrl: "http://10.0.2.2:4000"` and `extra.wsUrl: "ws://10.0.2.2:5000"`. Change if testing against staging.
- [ ] `npm run typecheck` clean.
- [ ] `npx expo-doctor` returns 18/18 passed.

Start the dev server: `npm run start`. Press `a` to launch Android. Expect Expo Go to load, then the Signalix login screen to render.

---

## 1. Auth — Login

- [ ] Login screen renders with identifier + password fields, "Sign in" button, "Create account" link.
- [ ] Empty submit shows nothing (button disabled or inert).
- [ ] Wrong password → red error text appears under the form. App does not crash.
- [ ] Correct credentials → spinner → lands on Chats tab. Bottom tab bar visible. Status bar light text on dark background.
- [ ] Close app and reopen → still logged in (session persisted via `expo-secure-store`).
- [ ] Sign out from Settings → returns to login screen, session cleared.

---

## 2. Auth — Register

- [ ] Tap "Create account" link → register screen.
- [ ] All four fields visible: email, username, displayName (optional), password.
- [ ] Submitting with missing email / username / password is inert.
- [ ] Submitting with a taken username → server error surfaces in red.
- [ ] Submitting with a valid new identity → lands on Chats tab. Auto-logged-in.
- [ ] Sign out → log back in with the just-created credentials → works.

---

## 3. Chat list

- [ ] Chats tab loads with the list of conversations. If empty, "No conversations yet." placeholder is centered.
- [ ] Each row shows: avatar circle with initials, peer name, last-message preview (truncated at ~80 chars), timestamp, unread badge (if any), green presence dot if peer is online.
- [ ] Pull-to-refresh → spinner shows, list re-fetches, returns to top.
- [ ] Tapping a row → pushes Conversation screen with the peer's name in the header.

---

## 4. Realtime — incoming message

Trigger from a second device (open the web app at `http://localhost:3000` in a browser, sign in as a different user, send a message to your mobile user):

- [ ] Mobile chat list re-orders to put the chat with the new message at the top.
- [ ] Last-message preview updates to the new text.
- [ ] Unread badge appears with count 1.
- [ ] Tapping the chat clears the badge.
- [ ] Receive a second message while inside the conversation → it appears at the bottom and scrolls into view. No unread increment (active chat already marks read).

---

## 5. Conversation — send + receive

- [ ] Composer at the bottom: text input + send button.
- [ ] Typing enables the send button.
- [ ] Sending a message → optimistic bubble appears immediately on the right side (sender bubble).
- [ ] Within ~1s, the bubble status icon transitions sent → delivered → read as the web client receives + opens the chat.
- [ ] Receiving a message from the web peer → appears on the left side, no delay > 2s.
- [ ] Scrollback: scroll up → older messages load (pagination). Spinner at top while loading.
- [ ] Tapping the back arrow returns to Chats tab. The chat retains the same position with no unread.

---

## 6. Conversation — auto-deliver + auto-read

- [ ] Open a chat with unread messages → the unread badge clears immediately.
- [ ] Server-side: web peer's last message moves to "read" within 1s (verify on the web client's status icon).
- [ ] Receive a new message while inside the chat → message arrives + immediately moves to read on the sender's side. No unread increment.

---

## 7. Group chats

- [ ] Group chats that exist on the web client appear in the mobile chat list with the group title (not a peer name).
- [ ] Tap a group → conversation loads, all member messages render with sender names above incoming bubbles.
- [ ] Send a group message → appears for the sender (right side), delivered + read updates fan out as web peers receive.
- [ ] Receive a group message → left-side bubble with sender name.
- [ ] No group management UI (out of scope for v0.16.0).

---

## 8. Plaintext degradation (the v0.16.0 trade-off)

- [ ] Direct chat: web user (E2EE-enabled) → mobile user. The mobile recipient sees `[encrypted]` placeholder text (or whatever the current fallback string is — the point is they do NOT see decrypted content, and the app does NOT crash).
- [ ] Direct chat: mobile user → web user. The web recipient sees the plaintext (not an envelope error).
- [ ] Direct chat: web user → web user. E2EE is intact (verify on both web clients).
- [ ] This degradation is documented in `MOBILE_ARCHITECTURE.md` — do not file a bug for it. v0.17.0 closes the gap.

---

## 9. Settings

- [ ] Settings tab shows: username, email, displayName, app version (should read "v0.16.0"), Sign out button.
- [ ] All fields are read-only in v0.16.0 (no edit UI yet).
- [ ] Sign out → confirmation? (current build has no confirm — clarify in QA whether this is expected). Then returns to Login screen, session blob cleared from `expo-secure-store`.

---

## 10. Network resilience

- [ ] Toggle Android airplane mode → composer should disable or queue (current build queues optimistically; check store behaviour).
- [ ] Turn airplane mode off → queued messages flush, status updates re-sync (mobile mirrors the web's v0.14.0 reconnect catch-up — `GET /chats/:id/messages?since=` should fire).
- [ ] Kill the realtime container (`docker stop signalix-realtime`) → mobile WS reconnects every 3s with backoff. App doesn't crash.
- [ ] Restart realtime → connection re-established, heartbeat (30s cadence) resumes.

---

## 11. Cold-start performance

- [ ] First launch after install: from tap to login screen should be under 4s on a modest emulator.
- [ ] After login (cached session): from tap to chat list should be under 2s.
- [ ] Pull-to-refresh full chat list: round-trip under 1s on localhost backend.

---

## 12. Known issues / non-goals

- iOS simulator support: untested in v0.16.0. Mobile works on iOS but is Android-first.
- Push notifications: not implemented. Mobile receives no banner when backgrounded. v0.18.0.
- Media / files / voice notes: not implemented. Sending or receiving a media message from the web side will likely show an empty bubble. v0.19.0.
- Calls: not implemented. v0.20.0.
- Edit / reply / forward / delete-for-everyone: web-only in v0.16.0. Mobile renders them as plain messages.
- Device verification UI (safety number): web-only. v0.22.0+.

---

## 13. Reporting bugs found during this checklist

When a step fails, capture:

1. The exact step number (e.g. "Section 5, item 3 — status never transitions past delivered").
2. Mobile logs: `adb logcat *:S ReactNative:V ReactNativeJS:V` while reproducing.
3. Metro bundler output if the bundle itself errored.
4. Whether the same flow works on the web client (so you can isolate mobile-specific bugs from server-side bugs).
5. The git SHA of `Signalix-mobile/HEAD` (`git rev-parse HEAD`) so the bug can be tied to a known build.

Add the report to a GitHub issue on `dennys9415/Signalix-mobile` (or to `docs/RELEASE_NOTES_v<next>.md` "Fixed" section if it's part of an in-flight release).
