# bluos-tui

Terminal client for [BluOS](https://www.bluesound.com/bluos/) players (Bluesound, NAD, …): browse the local music library in [fzf](https://github.com/junegunn/fzf) with vim keys, see the album's tracks with the playing one highlighted, and follow "now playing" with progress and volume in the footer.

> Unofficial. Not affiliated with Bluesound/Lenbrook.

## Install

### Requirements

- A BluOS player on the same local network as your computer.
- Music on the player's own library (USB drive or network share) – that is what bluos-tui browses.
- `bash`, `fzf` ≥ 0.63, `curl`, `ffmpeg` (for `ffprobe`), `gvfs-smb` (for `gio`), `iproute2`.

### Arch Linux (recommended)

```bash
git clone https://github.com/sukkergris/bluos-tui.git ~/Code/bluos-tui
cd ~/Code/bluos-tui
makepkg -si          # builds the package, installs the dependencies with pacman and installs bluos to /usr/bin
```

Update:

```bash
cd ~/Code/bluos-tui && git pull && makepkg -si
```

Uninstall:

```bash
sudo pacman -R bluos-tui-git
rm -rf ~/.config/bluos ~/.cache/bluos   # optional: config and year cache
```

### Other Linux distributions

Install the requirements with your package manager (check `fzf --version`; many distributions ship an older one – get a release from [GitHub](https://github.com/junegunn/fzf/releases)). Then link the script onto your `PATH`:

```bash
git clone https://github.com/sukkergris/bluos-tui.git ~/Code/bluos-tui
mkdir -p ~/.local/bin
ln -s ~/Code/bluos-tui/bin/bluos ~/.local/bin/bluos   # update later with git pull
```

### First run

```bash
bluos scan    # should list your player(s), e.g. "192.168.1.50 Living Room"
bluos         # opens the UI
```

On first run bluos uses the first player it finds and saves it in `~/.config/bluos/config` (see [Config](#config)). Pick another one on the Players tab (`gT`, then `enter`). The first start reads release years from every album (~0.3 s per album); later starts are fast.

### Troubleshooting

- **"No BluOS player found":** the computer and player must be on the same /24 network, and port 11000 must not be blocked. Set `host=<player IP>` in `~/.config/bluos/config` (the IP is shown in the BluOS app under Settings → Diagnostics or in your router).
- **Albums are not sorted by year:** the year is read from the player's SMB share. Check that `gio mount -a smb://<player IP>/shared` works and that `gvfs-smb` is installed. If your share has another name (`smbclient -N -L //<player IP>`), set `share=` in the config. Then delete `~/.cache/bluos/` so the years are read again.
- **fzf error about unknown options (`--no-input`, `--footer`):** your fzf is older than 0.63.
- **Empty album list:** the player has no local library, or it is still indexing – check the Library in the BluOS app.

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
