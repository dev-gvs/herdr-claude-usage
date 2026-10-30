# Claude usage

Your Claude plan limits in Herdr's status bar: the 5-hour and weekly usage as two small meters, each with a faint fill showing how much of the window has passed, so you can tell at a glance whether you are ahead of pace. Click it for a popup with every limit, its exact percentage and when it resets.

![Claude usage](preview.png)

This file is the brief: enough to build Claude usage from an empty folder. The files next to it are one build of it. If they don't work on your Herdr, or you want it different, build it again from here.

## What you see

- **The bar**, a block about 330px wide and 28px high, meant for the right end of a status bar. Background `chrome`, 12px side padding, 12px gaps, text 11.8px.
  - Herdr's Claude icon (`icons/claude.svg`, 14px).
  - Two meters, `5h` (the 5-hour session limit) and `7d` (the weekly limit): a `dim` label, an 80px wide, 5px high rounded track in `border`, and a 30px wide percentage in `muted`.
  - The used fill is clay: `#b98253` for 5h, `#856144` for 7d. At 80% and above it turns amber `#ffb347` (the percentage too), at 95% and above red `#ff5d6c`.
  - Behind it, a ghost fill in `dim` at 35% opacity shows the share of the window that has already elapsed. A lit fill past the ghost means you are using the limit faster than time passes.
  - Before the first fetch it shows `…`; with no data and an error, `no data`.
- **The popup**, 300px wide, sized by content, 14px padding, 14px gaps, text 12px in `muted`.
  - A header: the Claude icon (16px), "Claude usage" in 13px semibold `foreground`, and a "Refresh" button on the right (`dim`, `hover` background on hover).
  - One section per limit the account reports, including weekly limits scoped to one model ("Weekly Opus limit"): the name in `foreground`, the percentage in 18px semibold in the meter's color, a 6px meter, and "resets in 2h 14m, at 17:00" (under 24 hours) or "resets Monday at 09:00" in 11.5px `dim`.
  - A footer: "updated 14:05", or "fetch failed: HTTP 401 · data from 13:55" in `blocked`.

## How it behaves

- Fetches on start, then at most every 5 minutes (a 60-second timer checks when the last fetch was). Every copy shares one result through shared state `"claude-usage"`, so several bars or a bar and the popup don't fetch twice.
- Stale data (an error, or no successful fetch for 12 minutes) keeps the last numbers but greys the meters to `dim`, drops the ghost fill and dims the icon to 50%. The 12 minutes are one missed cycle plus slack, so a slow fetch doesn't flash the bar grey.
- Hover shows a tooltip with each bar limit's name, percentage and reset time, plus the last error.
- Click opens the popup (`kind: "popup"`, without `at`; Herdr places it above the bar). Refresh fetches at once. Esc closes it.
- Agent statuses play no part; there are no animations.

## Built from

- **One mod**, `mods/claude-usage/`, with `main.js` and `mod.json` (`place: "none"`, so it only shows where a layout puts it). No `keys.json`.
- **A block in a layout you own** (clone a shipped one first with `layout.clone`). For example a status row at the bottom of the work column:

  ```json
  { "id": "status-bar", "type": "row", "size": { "px": 28 }, "resizable": false,
    "children": [
      { "id": "status", "type": "block", "content": { "mod": "workspace-status" } },
      { "id": "claude-usage", "type": "block", "size": { "px": 330 },
        "content": { "mod": "claude-usage", "props": { "border": "top" } } } ] }
  ```

  Prop `border`: `"top"` or `"bottom"` draws a 1px `border` line on that edge, to match a bar beside it.
- **One view, two roles.** With `props().overlay` set it draws the popup (`openOverlay({ mod: "claude-usage", kind: "popup", width: 300, props: { details: true } })`), otherwise the bar. Only the bar runs the timer.
- **The data**: `GET https://api.anthropic.com/api/oauth/usage` with `Authorization: Bearer <token>` and `anthropic-beta: oauth-2025-04-20`, through `fetch`. Newer responses carry `limits: [{ kind, percent, resets_at, scope: { model: { display_name } } }]` with kinds `session`, `weekly_all` and `weekly_scoped`; older ones carry `five_hour` / `seven_day` with `utilization` and `resets_at`. Both are mapped to `{ kind, model, used, resets_at }`. Window lengths for the ghost fill: 5 hours for `session`, 7 days for the weekly kinds.
- **The token**: Claude Code's own OAuth login, read with `readTextFile` from `~/.claude/.credentials.json` and `~/.claude-work/.credentials.json` (`claudeAiOauth.accessToken`); when both exist, the one with the later `expiresAt` wins, since whichever Claude Code refreshed last holds the live token. The mod never stores, logs or refreshes the token; it sends it only to `api.anthropic.com`.
- **Herdr functions**: `home`, `readTextFile`, `shared`, `setShared`, `props`, `openOverlay`, `theme`; `image`, `div`, `h_flex`, `v_flex`, `.tooltip`, `.when`.

## Watch out

- **The mod reads files outside its folder and reaches the network**: Claude Code's credentials file, and Anthropic's API with that token.
- The usage endpoint is the one Claude Code's `/usage` uses; it is not a documented public API and its shape can change. Keep both response shapes.
- Credentials as a file is how Claude Code stores them on Linux and Windows. On macOS it keeps them in the Keychain, so there is no file and the bar shows `no data` ("no Claude Code login found"). A macOS build would read the token with `exec("security", ["find-generic-password", "-s", "Claude Code-credentials", "-w"])`.
- An expired token is reported, not refreshed: "token expired, use Claude Code to refresh it". Using Claude Code refreshes it, and the next fetch recovers.
- `~/.claude-work` is a second Claude Code config dir (`CLAUDE_CONFIG_DIR`); a missing one is skipped. Add or remove dirs in `token()`.
- The API reports resets a fraction of a second before the hour; round to the minute before printing, or "16:59" shows instead of "17:00".
- The bar's click handler stops the mouse-down so a `window_drag` status bar doesn't swallow the click.
- Times are 24-hour and local; days are English.

## Done when

- `herdr-app check mods/claude-usage` passes.
- With Claude Code logged in, the bar shows `5h` and `7d` meters with the same percentages as Claude Code's `/usage`.
- The ghost fill of `5h` grows over the session and resets with it.
- Clicking the bar opens the 300px popup with every limit and its reset time; Refresh updates the footer's time.
- Moving the credentials file away turns the meters grey within a fetch and the popup footer reads "fetch failed: no Claude Code login found"; putting it back recovers.
- `herdr-app status` shows no errors from the mod.
