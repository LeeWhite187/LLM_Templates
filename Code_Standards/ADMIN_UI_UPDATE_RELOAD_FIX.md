# Admin UI - "Reload after update" regression and fix (cross-project note)

**Date:** 2026-10-28. **Reference implementation:** Facility Dashboard Platform (FDP), commit `00037a8`.
**Audience:** any project that inherited FDP's admin-UI connectivity + version-skew banners (the SPA
shell with a live SignalR link, an HTTP heartbeat, and an `X-FD-Version` response header). Portable:
adapt the header name, API paths, and file names to the consuming project.

---

## Summary

After a service update, the SPA used to show a **yellow "Reload now" banner** that reloaded the tab
onto the new bundle. A later connectivity feature broke that: the update now first shows a **red
"service unreachable" banner**, and when the service returns the yellow reload prompt never appears,
so the operator is stranded on a stale bundle and must press **F5** by hand. The red banner's
**Retry** only re-probes the connection - it never reloads - so it looks like "Retry does nothing."

The reload mechanism itself was never broken. What broke is the *trigger*: the component that now
owns the moment the service returns throws away the version information that raises the reload banner.

---

## How to tell if your project is affected

You inherited this bug if all of these are true:

1. The shell shows a **version-skew banner** driven by an `X-FD-Version` (or equivalent) response
   header observed through an **HttpClient interceptor** (a first response sets the baseline; a later
   differing value raises the banner).
2. The shell also has a **connectivity heartbeat** that polls a lightweight endpoint (e.g. `/api/v1/me`)
   using a **raw `fetch`** (chosen to bypass the global loading indicator and HTTP caching).
3. That heartbeat updates the connectivity state but does **not** feed the version watch.

If the heartbeat is a raw `fetch` and does not call the version-observe function, you have the bug.

---

## Symptom (operator's words)

> "It used to be a yellow banner with a button to reload. Now the banner is red and asks to Retry, the
> Retry doesn't work, and I have to hit F5 to reload the page."

---

## Root cause

The shell has **two independent top banners**:

| Banner | Meaning | Trigger | Button |
|---|---|---|---|
| **Yellow** - "Reload now" | UI bundle is older than the running service (version skew) | an **HttpClient** response carrying a changed `X-FD-Version`, seen by the version interceptor | reloads the tab |
| **Red** - "Unreachable / Retry" | live link to the service is down | connectivity state `reconnecting`, fed by the SignalR lifecycle and the HTTP heartbeat | re-probes connectivity |

The connectivity heartbeat pings the service with a **raw `fetch`**, deliberately bypassing the
Angular HttpClient pipeline (so it does not trip the loading bar and cannot be served from cache).
That bypass is the trap: **the raw fetch never runs the version interceptor**, so the `X-FD-Version`
it receives is discarded.

Sequence during an update, post-regression:

1. Service stops for the upgrade -> HTTP + SignalR fail -> **red** banner.
2. Service returns on the **new** version -> the heartbeat's `fetch` succeeds -> red banner clears.
   But because the heartbeat bypassed the interceptor, **the new version was never observed**, so the
   **yellow "Reload now" banner never fires**. The tab is now running a stale bundle with no prompt.
3. The operator clicks the red banner's **Retry**, which only re-probes connectivity (and nudges
   SignalR). It never reloads the bundle, so nothing visibly changes. F5 is the only recovery.

Before the connectivity feature existed, the first HttpClient poll after the restart went through the
interceptor and raised the yellow banner directly. The connectivity layer intercepted that moment with
a red banner whose heartbeat does not carry version information forward.

---

## The correction (two parts)

### 1. Make the heartbeat feed the version watch (the real fix)

On a successful heartbeat response, read the version header off the `fetch` response and feed it to the
version watch, exactly as the HttpClient interceptor would. A **same-origin** `fetch` can read any
response header (no CORS allow-list needed), and the service already stamps the header on every API
response, so the value is present.

```ts
// AppComponent.startHeartbeat (authoring shell)
const ping = () => {
  fetch('/api/v1/me', { cache: 'no-store' })
    .then(res => this.zone.run(() => {
      if (res.status > 0) {
        this.conn.online();
        // Raw fetch bypasses the HttpClient version interceptor, so feed the version watch
        // ourselves - otherwise an update applied while the tab is parked clears the red banner
        // but never raises the yellow "Reload now" one.
        const version = res.headers.get('X-FD-Version');
        if (version) { this.versions.observe(version); }
      } else {
        this.conn.lost();
      }
    }))
    .catch(() => this.zone.run(() => this.conn.lost()));
};
setInterval(ping, 8000);
ping();
```

With this, an update applied while the tab is parked raises the yellow "Reload now" banner
automatically within one heartbeat interval, with no HttpClient traffic required.

**Rule to preserve:** any probe of the service that does not go through the HttpClient interceptor
(raw `fetch`, `navigator.sendBeacon`, a WebSocket handshake you can read headers from, etc.) **must
feed the version watch itself**. Do not "optimize" this line back out.

### 2. Give the red connectivity banner a Reload button (belt-and-suspenders)

A full reload always recovers, even in the rarer case where a parked tab genuinely cannot
re-establish its live connection after a restart (a dead keep-alive socket, a re-auth the background
request cannot satisfy silently). So the red banner should also offer Reload, making "a reload is one
click away" true regardless of which banner is showing.

```html
<div class="conn-banner" *ngIf="conn.state() === 'reconnecting'">
  <span>Lost connection to the service - trying to reconnect ...</span>
  <button (click)="retry()">Retry now</button>
  <button class="ghost" (click)="reloadNow()">Reload</button>  <!-- always recovers -->
</div>
```

`reloadNow()` is just `location.reload()`. Give the secondary button a lighter (outline) style so the
two actions are visually distinct.

---

## Why "Retry" alone could never fix it

`retry()` re-probes connectivity and nudges the live channel; it does not reload. Two consequences:

- A reconnect nudge that only restarts the live channel when it is fully *Disconnected* is a no-op
  during automatic reconnection (state *Reconnecting*), so it often does nothing visible.
- Even when the probe succeeds, it clears the red banner but cannot replace the stale JS/CSS already
  loaded in the tab. Only a document reload does that.

So "Retry" was never the control that recovers from an update - the reload banner was, and that was
the banner that stopped appearing.

---

## How to verify the fix

1. Load the admin UI; leave the tab open.
2. Deploy a new service build (or just restart the service on a build with a bumped version).
3. Expected: a brief red "unreachable" banner during the restart, then - within ~one heartbeat
   interval of the service returning - the **yellow "Reload now" banner appears on its own**. Clicking
   it loads the new bundle. No F5 required.
4. Separately, while the service is down, confirm the red banner now shows a working **Reload** button.

---

## Portability checklist for an inheriting project

- [ ] Confirm the service stamps a version header on every API response (adapt the header name).
- [ ] Confirm the version interceptor observes that header on HttpClient responses (baseline + skew).
- [ ] Find every **non-HttpClient** probe of the service (raw `fetch`, beacon, health ping). For each,
      read the version header and feed the version-observe function.
- [ ] Add a Reload button to the connectivity/"unreachable" banner.
- [ ] Verify with the real restart path (above), not just a dev reload.
