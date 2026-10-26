# dub-watch

A desktop notification when an anime on your MyAnimeList **On-Hold** list gets a
**finished English dub**, with a button that moves it to **Plan to Watch**.

Built for dub-only watchers who park shows On-Hold until the dub is done, and then
never notice when it is.

```
Dub complete: Frieren: Beyond Journey's End
All 28 episodes dubbed. Data: MyDubList (CC BY 4.0), AnimeSchedule.net
                                        [ Move to Plan to Watch ]
```

## How it decides a dub is finished

For every show on your On-Hold list:

1. **MyDubList** (primary, no key needed) says whether the show has an English dub,
   and whether that dub is partial.
2. **MAL** says whether the show itself has finished airing, and how many episodes it has.
3. **AnimeSchedule** (optional, needs a free token) adds two checks: a dub that is still
   releasing is held back, and the highest dubbed episode in its dub timetable must reach
   MAL's episode count.

| Result | Meaning | Notifies |
|---|---|---|
| no dub | MyDubList has no English dub | no |
| partial dub | MyDubList marks it partial, or AnimeSchedule counted fewer dubbed episodes than MAL lists | no |
| dubbed, show still airing | the Japanese broadcast is not finished on MAL | no |
| dub still releasing | AnimeSchedule has a dubbed episode still to come | no |
| dub complete (counted) | AnimeSchedule's dubbed episode count reaches MAL's count | yes |
| dub complete (listed) | MyDubList lists it as fully dubbed, no episode count available | yes |

Each show notifies once. Clicking the button sets it to Plan to Watch on MAL. Dismissing
the notification changes nothing on your list.

## Limitations

Read these before relying on it.

- **Linux desktop only.** Notifications use `notify-send` with an action button, which
  needs a notification server that supports actions (KDE Plasma and GNOME do). There is
  no Windows or macOS support.
- **English dubs only.**
- **MAL only, On-Hold only.** AniList and other list statuses are not checked.
- **Only as accurate as MyDubList.** It aggregates several sources and is updated daily,
  but it can miss a dub, list one too early, or not mark a partial dub as partial.
  dub-watch uses its `normal` confidence tier (at least two sources, or curated by hand).
  A newly announced dub can take a day or more to appear.
- **Without an AnimeSchedule token, "complete" is not verified.** A show that finished
  airing in Japan while its dub is still releasing weekly can be reported complete
  early, unless MyDubList marks it partial. There is also no episode count.
- **With a token, the count only covers dubs in AnimeSchedule's timetable.** Older dubs,
  home video (DVD/Blu-ray) dubs and some all-at-once releases never appear there and fall
  back to "listed". Split seasons can be numbered differently on AnimeSchedule than on MAL.
- **Depends on undocumented API behavior.** AnimeSchedule's authenticated responses lack
  the `dubExists` field (dub-watch uses the `dub-exists` filter instead), and looking up
  several MAL IDs in one request fails despite the docs, so it makes one call per dubbed
  show. If either service changes, dub-watch can break until updated.
- **Shows still airing in Japan are never complete**, even if the simuldub is caught up.
- **Runs at login, at most once a day**, not on a timer. `dub-watch --force` checks right away.
- **Once per show.** A dismissed notification is not repeated. To get it again, remove
  the show's ID from `notified` in the state file.
- **Not affiliated** with MyAnimeList, MyDubList or AnimeSchedule.

## Requirements

- Linux with a notification server that supports actions
- Python 3.11 or newer, with `requests` (`pip install --user -r requirements.txt`,
  or `python3-requests` from your distro)
- `notify-send` (libnotify 0.7.10 or newer)
- A public MAL list

## Setup

### 1. Install

```sh
git clone https://github.com/NealimeKenna/dub-watch.git
ln -s "$PWD/dub-watch/dub-watch" ~/.local/bin/dub-watch
```

### 2. Run setup and register a MAL API app (required)

```sh
dub-watch setup
```

Setup asks for your MAL username and the **login callback port** (default 8766). It
checks that the port is free and then prints the exact redirect URL for your MAL app,
for example `http://localhost:8766/callback`. Keep setup open and register the app:

Go to <https://myanimelist.net/apiconfig>, click **Create ID** and fill in:

| Field | Value |
|---|---|
| App Name | `Dub Watch` (anything) |
| App Type | `other` |
| App Description | e.g. `Personal script that checks my On-Hold list for finished English dubs.` |
| App Redirect URL | the URL setup printed, exactly (no trailing slash) |
| Homepage URL | your MAL profile, e.g. `https://myanimelist.net/profile/<username>` |
| Commercial / Non-Commercial | `non-commercial` |
| Name / Company Name | your name or username |
| Purpose of Use | `hobbyist` |

Then open the app, copy the **Client ID** and paste it into setup. Keep it private:
MAL's API agreement does not allow sharing it. Leave the client secret empty for app
type `other`.

If you change the port later with `dub-watch setup`, change the app's redirect URL to match.

### 3. AnimeSchedule token (optional)

Setup asks for this last; press Enter to skip. To get one, create an account on
<https://animeschedule.net>, go to **Settings > API**, create an application and copy its
**application token** (not the client secret). You can add it later by running
`dub-watch setup` again; it keeps your other answers.

Then log in to MAL once (for the Move button) and try a dry run:

```sh
dub-watch auth      # opens MAL in your browser once
dub-watch --dry-run # shows the status of every On-Hold show, changes nothing
```

### 4. Run at login

```sh
cp dub-watch/dub-watch.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable dub-watch.service
```

The service expects the command at `~/.local/bin/dub-watch`.

## Usage

```
dub-watch setup      write the config (MAL username and client ID, optional AnimeSchedule token)
dub-watch auth       one-time MAL login for the "Move to Plan to Watch" button
dub-watch            check (at most once per day), notify on finished dubs
dub-watch --force    check even if already checked today
dub-watch --dry-run  print the status of every show, change nothing
```

## What it stores

Everything stays on your machine:

- `~/.config/dub-watch/config.json`: username, client ID, AnimeSchedule token (mode 600)
- `~/.config/dub-watch/token.json`: the MAL login token, refreshed automatically (mode 600)
- `~/.local/state/dub-watch/state.json`: which shows were notified, last run date, and
  per-show dub episode counts from AnimeSchedule

Your MAL password is never seen or stored; login goes through MAL's own OAuth page.

## Data sources and credits

- [MyDubList](https://github.com/Joelis57/MyDubList) by Joelis57, licensed
  [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The dub data is downloaded
  at runtime and not redistributed.
- [AnimeSchedule.net](https://animeschedule.net) API, for dub timetables.
- [MyAnimeList](https://myanimelist.net) API, for your list and episode counts.

Notifications credit the sources they used.

## License

[PolyForm Noncommercial 1.0.0](LICENSE). You may use, modify and share dub-watch for
any noncommercial purpose (personal use, hobby projects, research, nonprofits). Selling
it, or using it in a commercial product or service, is not allowed.
