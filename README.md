# GSoC 2026 Final Report

## Multi-Screen Support for Jitsi Meet

| | |
|---|---|
| **Contributor** | Abhay Madan ([@codewithabhay10](https://github.com/codewithabhay10)) |
| **Email** | abhaymadan22@gmail.com |
| **Organisation** | Jitsi |
| **Project** | Multi-screen support: render meeting surfaces on additional displays |
| **Mentors** | Cosmin Timis ([@cosmintimis](https://github.com/cosmintimis)), Tudor Avram ([@quitrk](https://github.com/quitrk))|
| **Proposal** | [Drive link](https://drive.google.com/file/d/1A8-pW0nayhCvJ_Jp3L6_37HJkZJiNxRi/view?usp=sharing) |
| **Repository** | [jitsi/jitsi-meet](https://github.com/jitsi/jitsi-meet) |
| **Feature module** | `react/features/multi-screen/` |
| **Coding period** | 25 May to 24 August 2026 |

---

## Summary

Jitsi Meet rendered a meeting into exactly one browser window. On a room appliance
or a dual-monitor desk setup, the second display could only be used by manually
dragging a browser window onto it, which shows the whole meeting UI rather than the
content that actually deserves the extra screen.

This project adds a second-screen capability to Jitsi Meet: a meeting can now drive
one or more additional windows, each placed on a chosen physical display and each
rendering a chosen surface, the active-speaker stage, a screenshare, a tile grid of
every participant, the shared whiteboard, or a shared video. The windows are driven
by the iframe External API so that embedders and room appliances can control them
programmatically, and, as of the final PR, directly from the meeting UI so that an
ordinary user on a laptop with a monitor can use the feature without writing any
integration code.

Five of my pull requests have been merged into `master`, totalling **+3,324 / -426
lines** across 50 file changes. The last of them, the in-app triggers (#17666), was
merged on 19 August after four rounds of review, which completes the feature itself.
Two more are finished from my side and sit in the review queue, neither of them
production code: WebdriverIO specs and a manual test checklist (#17715), and the
handbook documentation for the whole External API surface (jitsi/handbook#683). The feature module on `master` is now **3,413 lines
across 22 files**.

---

## Goals from the proposal

| # | Goal | Outcome |
|---|---|---|
| 1 | Let a meeting render a second surface on a second physical screen | Done, merged |
| 2 | Support multiple layouts on that surface (a featured stage, a tile grid) | Done, merged (#17581) |
| 3 | Make the second screen controllable in a mouseless / kiosk context | Done, merged, via the External API, plus in-app triggers for mouse users (#17666) |
| 4 | Allow non-video content (e.g. the whiteboard) on the second screen | Done, merged (#17581, #17598), and extended to the shared video (#17615) |

Goal 4 went beyond what the proposal asked for: the shared video was added after the
midterm at the mentors' request, and it is a second class of non-video content with
its own player and playback synchronisation rather than a variation on the
whiteboard.

---

## The problem

A second display is useful in a meeting for reasons that differ from more screen
area in the same window:

- A room appliance driving a TV wants the active speaker large and nothing else, no
  toolbar, no filmstrip, no chat.
- A presenter wants the shared screen on the projector while keeping the participant
  grid and controls on the laptop.
- A user watching a shared video wants it on the big screen while the meeting stays
  on the small one.

None of this is achievable by resizing the existing window, because the surfaces
have to be selected and laid out independently of the main meeting UI. It also
cannot be solved by simply opening a second copy of the meeting: that would join the
conference twice, duplicate the media subscriptions, and play the audio twice.

The design constraint that shaped almost every decision is that **the second window
must be a view onto the existing conference, not a second conference**.

---

## What was built

This section describes the feature as it now stands, which includes the base it
started from. That base is [#17527](https://github.com/jitsi/jitsi-meet/pull/17527)
by Emil Ivov: the config gate, the redux model, the External API command and its
three events, and opening and placing the window. Each part below says whether it
came from that base or from this project, so the boundary is explicit.

### Architecture

**A config gate.** *(base)* The feature is off unless a deployment opts in:

```js
secondScreen: {
    enabled: false
}
```

It is Chromium-only (it depends on the Window Management API) and intended for
managed or kiosk room appliances, which must also delegate
`allow="window-management; fullscreen"` to the iframe and grant the
window-management and automatic-fullscreen permissions.

**Redux as the single source of truth.** *(base; this project added the `setAt`
timestamp)* State holds a map of second-screen entries keyed by id. Each entry
records what the window should render, which physical screen it belongs on, when it
was last configured, and (once open) an opaque live handle to the real `Window`.
Middleware reconciles actual browser windows to that state, so there is exactly one
place that describes what should be on screen and one process that makes reality
match it.

**The External API as the control plane.** *(base)* Everything goes through one
command and three events:

| Kind | Name |
|---|---|
| Command | `setSecondScreen` (`set-second-screen`) |
| Event | `secondScreenSourceChanged` |
| Event | `secondScreenClosed` |
| Event | `secondScreenError` |

The in-app UI triggers added in the final PR dispatch the *same* action the API
dispatches, rather than taking a parallel path. This was a deliberate choice: it
keeps one control plane, so an embedder observing the events sees in-app activity
too, and there is only one code path to reason about when something goes wrong.

**Rendering by React portal.** *(this project, #17547)* The second window's content is rendered by the main
meeting's React tree, through a portal into the popup's document, with a per-window
Emotion cache so styles are injected into the right document. This is what makes the
second window a view rather than a second app: it shares the meeting's redux store,
its media tracks, and its component code, and it costs no extra conference
connection.

**Source selection is by name, never by stream.** *(schema from the base, which
shipped `stage` and `screenshare`; this project added `tile`, `whiteboard` and
`sharedvideo`)* An iframe embedder has no access to the media tracks, since those
live inside the meeting document. So a source names a *role* and optionally a
*participant*, and the feature resolves that internally to a live track:

```ts
interface ISecondScreenSource {
    media?: 'camera' | 'desktop';
    participant?: string;
    role?: 'stage' | 'screenshare' | 'tile' | 'whiteboard' | 'sharedvideo';
}
```

Roles stay live: `stage` follows the active speaker, `tile` follows conference
membership.

### Surfaces supported

| Role | What the window shows | Added by |
|---|---|---|
| `stage` | Active-speaker stage, with optional filmstrip | base; layout by #17581 |
| `tile` | Tile grid of every participant | #17581 |
| `screenshare` | A screenshare full-bleed, selectable by participant when several are live | base; per-participant selection by #17666 |
| `whiteboard` | The shared collaborative whiteboard | #17581, #17598 |
| `sharedvideo` | The video shared into the meeting (YouTube or a direct link) | #17615 |

---

## Work completed

| PR | Title | Merged | Size |
|---|---|---|---|
| [#17547](https://github.com/jitsi/jitsi-meet/pull/17547) | Render second screens via a React portal | 2026-06-30 | 8 files, +322/-186 |
| [#17581](https://github.com/jitsi/jitsi-meet/pull/17581) | Stage/tile/gallery layouts, and the whiteboard as a source | 2026-07-09 | 13 files, +1106/-102 |
| [#17598](https://github.com/jitsi/jitsi-meet/pull/17598) | Gate the whiteboard second screen on `isWhiteboardOpen` | 2026-07-09 | 1 file, +11/-4 |
| [#17615](https://github.com/jitsi/jitsi-meet/pull/17615) | Shared video as a second-screen source | 2026-07-31 | 11 files, +657/-40 |
| [#17666](https://github.com/jitsi/jitsi-meet/pull/17666) | In-app triggers for the second screen | 2026-08-19 | 17 files, +1228/-94 |
| [#17715](https://github.com/jitsi/jitsi-meet/pull/17715) | iframe API specs and a manual test checklist | **open** | 2 files, +368/-0 |
| [jitsi/handbook#683](https://github.com/jitsi/handbook/pull/683) | Document the second-screen API | **open** | 3 files, +198/-0 |

**Merged total: 50 file changes, +3,324 / -426.**

One further PR, [#17562](https://github.com/jitsi/jitsi-meet/pull/17562) (stage and
tile layouts), was closed unmerged on 2026-07-09 after review feedback; its content
was reworked and landed as part of #17581 the same day.

The project builds on
[#17527](https://github.com/jitsi/jitsi-meet/pull/17527) by Emil Ivov, merged
2026-06-23, which is not my work. It established the feature module, the config
gate, the External API command and events, and the ability to open a correctly
placed, fullscreen window on a chosen display via the Window Management API.
Everything below is the rendering, layout and control work built on that base.

### Before the pivot

The project did not start on top of #17527. It started from the inside of the app,
driven by user clicks: a secondary browser window sharing the main window's redux
store and WebRTC tracks, with no duplicate connection and no IPC, its own window
lifecycle, and two layouts. That was developed as a stack of pull requests on my
fork, each building on the last:

| Link | Summary |
|---|---|
| `feat/multi-screen-redux-foundation` | The actionTypes, actions, reducer and middleware registration the stack built on |
| [#3](https://github.com/codewithabhay10/jitsi-meet/pull/3) | Window management actions and middleware |
| [#4](https://github.com/codewithabhay10/jitsi-meet/pull/4) | Active-speaker and gallery secondary-window views |
| [#5](https://github.com/codewithabhay10/jitsi-meet/pull/5) | Config integration, last-layout memory, browser-fallback hint |
| [#6](https://github.com/codewithabhay10/jitsi-meet/pull/6) | Stage and Tile layouts with click-to-pin |

That went upstream as [#17434](https://github.com/jitsi/jitsi-meet/pull/17434). I
also built a secondary toolbar for the second window (microphone, camera and hang-up
with auto-hide) and secondary notifications mirrored onto the second screen, neither
of which was submitted upstream.

**The pivot.** In a weekly sync the team found that this overlapped with an official
pull request, [#17527](https://github.com/jitsi/jitsi-meet/pull/17527) by Emil Ivov,
which approached the same feature from the External API side. That is the control
plane a kiosk actually needs, since it has no mouse. Rather than ship a competing
implementation, we agreed that Emil's PR would land first and I would rebuild the
rendering layer on top of it, in the same feature folder, with PRs directly against
`jitsi/jitsi-meet`. I reviewed his PR before it merged, contributed a
dependency-declaration fix to it, and closed #17434 in favour of building on it.

The prototype was not wasted. #17527 rendered a single `<video>` per window and had
no UI at all; the layouts, the pinning behaviour and the visual design from the fork
work are what grafted onto it, and the exploration is where I learned the redux
registry architecture, the filmstrip and participant model, and track handling.

### First half

The first half turned the mechanism into something that could actually render a
meeting. #17547 replaced the initial rendering approach with React portals, which
is what made the second window share the meeting's store and tracks instead of
duplicating them. #17581 added the real layouts (stage, tile, gallery, and the
stage-tile component) and the whiteboard source, and #17598 fixed a gap where a
whiteboard second screen could be opened while the whiteboard itself was closed.

### Second half

#17615 added the shared video as a source, which turned out to be the hardest single
piece of the project (see below). #17666, merged on 19 August, added the in-app
triggers: a "Show on second screen" entry in the local, remote and fake-participant
context menus, a hover icon on the screenshare and shared-video thumbnails that have
no context menu of their own, a toggle-off path, user-visible failure notifications,
and screen selection that fills a free external display first and then reuses the
least recently targeted window. `tile` and `stage` stay External-API-only by
decision rather than by omission, for the reason given under design decisions below.

The last stretch was review rather than new surface, and four rounds on #17666
changed the feature in ways worth recording. Screen occupancy stopped being read
from the index a window was opened with and is now derived from where the windows
actually are, which is what makes docking and undocking displays behave; the window
id stopped encoding a screen index once occupancy became positional, since two
answers to the same question drift apart. An in-app second screen now closes itself
when what it was sent leaves the meeting, because every trigger lives on the thing
it sends, so a screenshare that stops takes its own off switch with it. 

Two bugs
came out of writing the tests rather than out of review: an unanswered
window-management prompt could leave an open pending forever with nothing reported
to anyone, and the window was being registered only after that prompt was answered,
which made a cancel during it silent. The second of those was a regression against
`master` and is fixed by registering the window as soon as its page loads, leaving
placement as a tail that holds no lock. The final round then found the consequence
of that change: a refused permission now reaches the teardown with the window
already registered, so it reported `secondScreenClosed` on top of
`secondScreenError`, and an embedder that reopens on a close would loop on it.
Detaching the handle before removing the entry drops the extra event and unmounts
the portal while the window is still open, which is the ordering the middleware
documents.

The rest of the period went into the two things the feature was missing
rather than into more feature: #17715 adds WebdriverIO specs for the External API
paths plus a 44-case manual checklist for everything CI cannot reach, and
jitsi/handbook#683 documents the command, the events, the error codes and the
config option, none of which had any handbook coverage at all.

---

## Design decisions and challenges

**Keeping the popup blocker satisfied.** Placing a window on a specific display
requires `getScreenDetails()`, which is async and prompts for permission the first
time. Awaiting it inside a click handler spends the click's user activation, which
Chromium expires after five seconds, so the browser blocks the popup. The base
awaited it on every open; this project caches the `ScreenDetails` object and
pre-loads it on `CONFERENCE_JOINED` when the permission has already been granted
(which never prompts on its own), so an ordinary send stays synchronous inside the
click's own task.

That still left the first send on a profile that had never granted the permission,
where the prompt sits between the click and `window.open` for as long as the user
takes to answer it. Review of #17666 raised this, and the answer was to stop
depending on the ordering: the window is now opened before anything that awaits,
unplaced, and moved onto its target screen once the permission is answered. No path
now puts a prompt between the click and the open.

Deferring the placement had a consequence I did not see until the next review round.
The open still held the window's redux entry and its in-flight lock while it waited
on the answer, so for as long as the prompt was up the window existed but had no
handle in state, and a cancel in that gap found nothing to close: it reported
nothing and left the trigger reading "Remove from second screen". Three separate
findings across two rounds all lived in that one gap, which was the argument for
closing it rather than guarding each case. The window is now registered as soon as
its page loads, which is the point where it can actually be rendered into and
closed, and the permission wait became a tail that holds no lock and re-checks that
it still owns the window before placing it. The trade is real and deliberate: the
content is now built before the permission is known, so a refusal tears down a live
window rather than an empty one. On a profile that has already denied the permission
this shows as a window that opens and closes again: correct and reported, but it
flashes. The tidy fix is for the preload to cache a known-denied state so the open
never starts, rather than a fast-fail path inside the open, which would reintroduce
exactly the kind of per-case guard the registration change removed. It is recorded
at case A8 of the manual checklist.

**Styling across a window boundary, and a bug that only existed in production.** The
portaled components are MUI/tss-react, so their styles have to reach the popup's
document. My first approach copied the meeting's stylesheet nodes into the popup. It
worked perfectly under `make dev` and would have failed silently in production:
Emotion's production "speedy" mode inserts rules through `insertRule` into `<style>`
tags that are empty in the DOM, so node-cloning copies nothing. A reviewer caught it
before it shipped. The fix is a per-window Emotion cache whose container is the other
window's `head`, which is the correct pattern for MUI across realms. The lesson that
stuck was to test the production build rather than only the dev server.

**Making embeds work in the popup.** A popup opened with `about:blank` has no
meaningful referrer, which breaks embeds such as the YouTube IFrame API. The window
is therefore navigated to a small shell page (`static/secondScreen.html`) and the
portal is mounted into that document once it has loaded.

**Verifying the right page loaded.** A `load` event fires for *any* served response,
including a proxy error or a 404. The shell page carries a
`<meta name="jitsi-second-screen">` marker, and the handle is only built if the
marker is present; otherwise the window is closed and the failure reported. This came
out of review and replaced a weaker check that treated a bare load as success.

**Window lifecycle robustness.** This absorbed three rounds of review on #17615, and
is the part of the code that changed most as a result. The final shape guarantees
that every failure exit performs the same teardown (close the window, notify the
embedder, drop the redux entry), that a window closed mid-load reports the same
`secondScreenClosed` event as one closed later rather than an error, that no
interval or timeout leaks, and that a second request for the same id arriving while
the first is still opening cannot orphan listeners or duplicate roots. The final
review round also removed an unreachable fast path whose only possible match was a
stale outgoing document.

**Shared video ownership.** The shared video player is effectively a singleton: its
constructor registers itself globally, and mounting a second instance in the popup
caused an ownership collision with the main window. A naive second mount also
dispatched side effects meant only for the main window (docking the toolbox, smart
audio mute), and both windows would have played the soundtrack. The solution was a
follower mode: the second-screen player never claims ownership, never fires update
events, never mutes or docks, hides its controls, and simply follows redux. For
YouTube specifically, the IFrame API's listener lives on the opener while the iframe
lives in the popup, and a video opened while paused needed `seekTo(time, true)` to
land on the right frame rather than the thumbnail.

**Absence is not proof of departure.** Every in-app trigger lives on the thing it
sends, so when that thing leaves the meeting the window loses its off switch and has
to close itself. The last review round pointed out that the check behind this cannot
tell a departure from a flap: a participant whose presence blinks, or whose client
rejoins the room, is removed and re-added, and a check that runs in between sees the
same absence as a real leave. Closing is one-way, because the send was a one-shot
action whose trigger went with the tile, so being wrong costs a window the user
cannot get back, while being slow costs a few seconds of a window showing an avatar.
A participant-backed source now gets a short grace period to come back before its
window closes. The whiteboard and the shared video do not wait, because those only
stop when someone stops them, which is deliberate and should not lag. I disagreed
with the mechanism that was proposed, since the bulk participant teardown it cited
requires leaving the room, which ends the conference and resets every window anyway,
so no grace period could reach it. The conclusion was right for the case that is
actually reachable, and I said both.

**Which roles get a trigger in the UI.** An in-app trigger has to live on
something. `screenshare`, `whiteboard` and `sharedvideo` each have an object on
screen to hang it on, and a participant's own tile is the obvious place to send that
participant. `tile` and `stage` have neither: they describe the meeting as a whole
rather than anything the user can point at, so a trigger for them would have to
invent a home in the overflow or layout area and would mean something different from
every other entry in the same menu. We settled on leaving those two roles to the
External API, which is where a kiosk drives them from in any case. The in-app
triggers cover what a user can point at, and the API covers the meeting-wide modes.

**Choosing which window to reuse.** With more displays than windows, a new send
should take a free external screen. Once every screen has one, it should reuse the
least recently targeted window rather than an arbitrary one, and it should never
take over a window the embedder opened through the API. Recency is a timestamp
stamped in the action creator, which keeps the reducer pure.

Which screen counts as free turned out to be the harder half, and review drove it
through two shapes. Reading it from the index a window was opened with is wrong the
moment anything moves: the browser renumbers its screen list when a display is
docked or undocked, and a window is never re-placed after it opens, so the stored
index quietly stops describing reality. It is now derived from where each window
actually is, by locating its centre point in the multi-screen coordinate space, and
a position that cannot be read counts as occupying its screen rather than as free,
so nothing is ever opened on top of a window that is already there. Making occupancy
positional then made the window id's embedded screen index a second, conflicting
answer to the same question, so the id stopped carrying one at all.

---

## Current state

**The feature is complete and merged.** #17666, the last piece of implementation,
went into `master` on 19 August as `ec0efb6b`, after four rounds of review. Every
goal in the proposal is now on `master`, reachable both through the External API and
from the meeting UI.

Two PRs are finished from my side and sit in the review queue, neither of them
production code:

- **#17715**, the WebdriverIO specs and the manual checklist. Reviewed once, both
  findings fixed, awaiting a second look.
- **jitsi/handbook#683**, the External API documentation. Reviewed once, all four
  findings fixed. It was deliberately held for #17666, because the page states the
  contract as that branch leaves it, so the merge on 19 August unblocked it.

All work passes the project gates: `tsc:web`, `tsc:native`, `eslint
--max-warnings 0`, and `lint:lang`. The feature has been exercised by hand
throughout the coding period on Chromium with a second physical display, which is
how most of the behaviour described above was arrived at in the first place.

One caveat I would rather state than leave implied: CI has never run on any of this.
Fork PRs sit at `action_required` until a maintainer approves the workflow, and that
did not happen, not across four rounds of review on #17666 and not on the merge
commit, which still reports zero check runs. Every check on this work has therefore
been a local gate run, a reviewer reading the code, or my own hands on two displays.
The 44-case checklist in #17715 is there so the next person gets that coverage
without having to rediscover what is worth checking.

---

## Benefits to Jitsi Meet

Spot-style kiosks and room appliances can drive a second display with a real meeting
layout, a featured stage or a tile grid, and with non-video content such as the
whiteboard or a shared video, all controlled remotely through the External API with
no mouse involved. That was the motivating use case, and it is served entirely
through a documented command rather than through anything Spot-specific in the
meeting code.

The same rendering layer serves the ordinary desktop user with a second monitor. As
of #17666 they do not need an embedder at all: the triggers in the meeting UI
dispatch the same action the API dispatches, so the two audiences share one code
path and one set of events.

Everything is behind a config flag that is off by default, so no existing deployment
changes behaviour until it opts in.

---

## Links

- Feature module: [`react/features/multi-screen/`](https://github.com/jitsi/jitsi-meet/tree/master/react/features/multi-screen)
- External API: [`modules/API/API.js`](https://github.com/jitsi/jitsi-meet/blob/master/modules/API/API.js), [`modules/API/external/external_api.js`](https://github.com/jitsi/jitsi-meet/blob/master/modules/API/external/external_api.js)
- Config option: `secondScreen` in [`config.js`](https://github.com/jitsi/jitsi-meet/blob/master/config.js)
- My PRs, merged: [#17547](https://github.com/jitsi/jitsi-meet/pull/17547), [#17581](https://github.com/jitsi/jitsi-meet/pull/17581), [#17598](https://github.com/jitsi/jitsi-meet/pull/17598), [#17615](https://github.com/jitsi/jitsi-meet/pull/17615), [#17666](https://github.com/jitsi/jitsi-meet/pull/17666) (in-app triggers)
- My PRs, open: [#17715](https://github.com/jitsi/jitsi-meet/pull/17715) (tests and manual checklist), [jitsi/handbook#683](https://github.com/jitsi/handbook/pull/683) (documentation)
- My PRs, closed: [#17562](https://github.com/jitsi/jitsi-meet/pull/17562), superseded during the pivot
- The base this builds on, by Emil Ivov: [#17527](https://github.com/jitsi/jitsi-meet/pull/17527)
- Pre-pivot prototype: [#17434](https://github.com/jitsi/jitsi-meet/pull/17434) (closed deliberately), fork PRs [#3](https://github.com/codewithabhay10/jitsi-meet/pull/3), [#4](https://github.com/codewithabhay10/jitsi-meet/pull/4), [#5](https://github.com/codewithabhay10/jitsi-meet/pull/5), [#6](https://github.com/codewithabhay10/jitsi-meet/pull/6)

---

## Acknowledgements

Thank you to Cosmin Timis and Tudor Avram for the guidance and the reviews, which
were consistently specific enough to be worth more than the code they were about. To
Emil Ivov for the External API foundation this project builds on, and for the design
decision that made the pivot the right call rather than a setback. And to Jaya Allamsetty and the Jitsi
community.
