# AGENTS.md — handball-overlay (FRH Live Overlay Suite)

Instructions for any AI coding agent or human developer. This repo is PUBLIC: never commit private
room names, keys or links. Security model and open to-dos: `SECURITY.md`.

## What it is
Live scoreboard / graphics overlay for handball broadcasts (OBS / Yolobox browser source), driven from
a phone. Match data comes from sportinfocentar2.com (the feed behind frhlive2.com).

## Pages (static HTML, no build step)
- `index.html` — start page: the user types a name → a room id is derived → it shows the control and
  output links.
- `control.html` — the operator's panel (phone): fetches the match feed from sportinfocentar, picks
  scenes, colours, language, opacity; writes everything to Firebase `rooms/{room}`.
- `output.html` — the overlay itself (OBS/Yolobox): only reads `rooms/{room}` (no direct feed access).

## Hosting
- GitHub Pages: https://pakotz-coder.github.io/handball-overlay/ — deploy = push to `main`.
- Firebase Realtime Database (project `handball-overlay`, europe-west1). Rules live in
  `database.rules.json` and are published by hand in the Firebase console (Realtime Database → Rules).
  The Firebase web config in the HTML is public by design; access is controlled by the rules.

## Rules
- The room id is the only secret ("room = password"): never log or commit real room names.
- Be gentle with sportinfocentar: the control panel polls only while a match is on air; no bulk requests.
- Data from the feed is rendered as text (escaped), never as HTML; avoid `new Function()`/`eval` on
  third-party data (see `SECURITY.md`).
- Test locally by opening the HTML files through any static server (e.g. `npx serve .`) and a test room.
- Commit messages: Romanian or English, explain why.
