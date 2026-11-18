# bluos-tui

Terminal client for [BluOS](https://www.bluesound.com/bluos/) players (Bluesound, NAD, …): browse the local music library in [fzf](https://github.com/junegunn/fzf) with vim keys, see the album's tracks with the playing one highlighted, and follow "now playing" with progress and volume in the footer.

> Unofficial. Not affiliated with Bluesound/Lenbrook.

## Install

Arch Linux:

```bash
git clone https://github.com/sukkergris/bluos-tui.git
cd bluos-tui && makepkg -si
```

Other systems: put `bin/bluos` on your `PATH` and install the dependencies: `bash`, `fzf` ≥ 0.63, `curl`, `ffmpeg` (`ffprobe`), `gvfs-smb` (`gio`), `iproute2`.

## Usage

```bash
bluos              # open the UI
bluos play|pause   # start / pause
bluos next|prev    # next / previous track
bluos vol [N|+N|-N]
bluos status       # what is playing
bluos scan         # BluOS players on the local network
bluos albums       # albums in the library
```

| Key | Action |
|---|---|
| `j` / `k`, `gg` / `G`, `ctrl-d` / `ctrl-u` | Move |
| `l` / `h` | Albums ⇄ the album's tracks |
| `gt` / `gT` | Tabs: Albums → Tracks → Players |
| `/` · `tab` · `esc` | Search · cycle album/artist/track (all tracks) · back to key mode |
| `?` | Show/hide shortcuts |
| `enter` | Play album, or the album from the selected track · choose player |
| `r` · `s` · `R` | Players tab: rescan · toggle shuffle · cycle repeat (queue → track → off) |
| `space` · `n` / `p` · `+` / `-` | Pause · next/previous · volume ±5 |
| `q` | Quit |

## Config

`~/.config/bluos/config` (created on first run with the first player found, and updated when you choose a player on the Players tab):

```bash
host=192.168.1.50   # player IP; reserve it in your router's DHCP
share=shared        # the player's SMB share (used to read release years)
```

`BLUOS_HOST` in the environment overrides `host`.

## How it works

- **API:** the BluOS HTTP API on port 11000. The library is only served per letter section via `/Browse` (`/Albums` answers empty), so sections are fetched one by one.
- **Sorting:** albums are sorted by artist, release year and album. BluOS does not expose the year, so it is read with `ffprobe` from the first track's tags via the player's own SMB share (mounted with `gio mount`; `/var/mnt/SMB-*/` in `file=` maps to the share root). That takes ~0.3 s per album, so years are cached in `~/.cache/bluos/years-<host>.tsv`; delete it to re-read (e.g. after fixing tags). Albums without a year sort last for their artist.
- **Structure:** the script calls itself (`_enter`, `_key`, `_scope`, `_tracks`, `_preview`, `_now`, …) from fzf key bindings. State (active tab, chosen album, position, album list) lives in `$XDG_RUNTIME_DIR/bluos-<pid>/`. The chosen player is kept in the state file `host`, so all sub-calls and the ticker follow along when it changes.
- **Tabs and `g` combos:** fzf only knows single keys, so `g`/`t`/`T`/`h`/`l` go through `_key`, which prints fzf actions (`transform`). `g` stores a timestamp; if `g`/`t`/`T` follows within 1 s it becomes `gg`/`gt`/`gT` (like vim's `timeoutlen`).
- **Search:** rows have artist and album in separate fzf fields (separated by `‒`, U+2012, which rarely appears in names), so `tab` can switch `--nth` between album (field 2) and artist (field 1). "track" switches to a list of all tracks, fetched from the Song sections the first time and cached. The query is kept when switching.
- **Single track:** `enter` on a track uses its `autoplayURL`, which starts the album from that track and plays the rest afterwards.
- **No `--track`:** global tracking blocks the action chain when the list switches between albums and tracks. Instead `--id-nth=3` + `track-current` is used only when the ticker reloads the track list.
- **Live updates:** a background loop (`_ticker`) fetches `/Status` every second and sends `change-footer` to fzf via `--listen` on a Unix socket; when the track changes it also sends `refresh-preview` (and `reload` of the track list on the Tracks tab). The loop and socket are removed on exit (`trap EXIT`).
- **Highlight:** the playing track is found by matching `<fn>` from `/Status` against `file=` in the track's `playURL` (titles can repeat on an album).

## TODO

- [ ] Test the playlist API before building on it: `Add?service=LocalMusic&file=...&where=last` (append to queue) and `Save?name=Test` (save queue).
- [ ] `a`: add album/track under the cursor to the queue (`playnow=1` → `where=last`).
- [ ] `w`: save the queue as a playlist (`Save?name=…`).
- [ ] **Playlists** tab: list and play saved playlists (browse `BluOS:`).
- [ ] **Queue** tab (`/Playlist`).
- [ ] Not tested yet: `s`/`R` (shuffle/repeat) and switching to another player.
- [ ] Alternative: `.m3u` files on the SMB share are imported by BluOS as playlists.

## License

MIT
