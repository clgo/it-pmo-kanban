# Hook: meeting-reminder (v2 website)

| | |
|---|---|
| **Trigger** | The visitor has been on the page for 10 seconds (`MEETING.delayMs = 10000`). |
| **Action** | Show a modal alert dialog: *IT meeting next week · 10:00 AM · Room 888 · Attendance is mandatory.* |
| **Code** | `index.html` → `initMeetingHook()` (script section "HOOKS (v2)"); markup `#meeting-backdrop`; styles section 8. |
| **Design** | Reuses the board's tokens: gradient hero tile, `--layer-2` fact tiles, red mandatory notice, pill button, spring entrance. Light/dark follows the system theme. |

## Behaviour

- Fires once per page load. Nothing is stored (the project forbids localStorage/cookies), so it appears again after a refresh.
- If the Add Task modal is open when the timer fires, the hook waits (re-checks every second) instead of stacking dialogs.
- `role="alertdialog"`, focus moves to **I will attend**, Tab is held inside, Esc or the button closes it and returns focus to where the user was.
- Motion respects `prefers-reduced-motion`.

## Change the meeting

Edit the text in the `#meeting-backdrop` markup (`#meeting-title`, `#meeting-time`, `#meeting-room`) and `MEETING.delayMs`.
