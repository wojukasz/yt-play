# yt-play

Simple, deliberately not overengineered, audio-only YouTube playback
from the terminal. Streams through `mpv` (no video window) with
`yt-dlp` resolving the URL and auto-reconnects if the stream drops.

One file, no build step: drop `yt-play` on your `PATH`, run it once to
generate a config, edit in your own stations, done.

## Requirements

| Package | Purpose | Needed |
| --- | --- | --- |
| [`mpv`](https://mpv.io) | playback | always |
| [`yt-dlp`](https://github.com/yt-dlp/yt-dlp) | resolves the stream URL | always |
| `procps` (`pkill`/`pgrep`) | `stop`/`status` | always, but almost certainly already installed |
| [`mpv-mpris`](https://github.com/hoyon/mpv-mpris) | play/pause via media keys | optional, Linux only |

```sh
# Arch
sudo pacman -S mpv yt-dlp mpv-mpris

# Debian/Ubuntu
sudo apt install mpv yt-dlp mpv-mpris

# Fedora (mpv-mpris ships in RPM Fusion Free, enable it first)
sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
sudo dnf install mpv yt-dlp mpv-mpris

# macOS (Homebrew) - no MPRIS on macOS (it's D-Bus/Linux-only), skip it
brew install mpv yt-dlp
```

`mpv-mpris` autoloads from `/etc/mpv/scripts/` once installed, no
config needed. It puts mpv on the MPRIS D-Bus interface, so `playerctl`
and your desktop's `XF86AudioPlay`/`XF86AudioPause` keys can pause and
resume it exactly like Spotify or a browser tab. `yt-play stop` from
the terminal still works without it (kills the stream outright) — it's
just pause/resume that's keyboard-only, not stop. On macOS, use mpv's
own `--input-ipc-server` for scripted pause/resume instead.

## Install

```sh
curl -o ~/.local/bin/yt-play https://raw.githubusercontent.com/wojukasz/yt-play/main/yt-play
chmod +x ~/.local/bin/yt-play
```

Or clone and symlink:

```sh
git clone https://github.com/wojukasz/yt-play.git
ln -s "$PWD/yt-play/yt-play" ~/.local/bin/yt-play
```

## Usage

```sh
yt-play lofi       # play a configured station
yt-play synth
yt-play <url>      # play any YouTube URL, audio only
yt-play status
yt-play stop
```

Starting a new stream automatically stops whatever was already playing.
If the stream drops (network blip, stall), it reconnects on its own
instead of going silent.

## Configuring

First run creates `$XDG_CONFIG_HOME/yt-play/stations.conf` (usually
`~/.config/yt-play/stations.conf`) with two example stations and a
commented-out cookies setting. Add stations one per line:

```
name=youtube-url
```

Then `yt-play <name>` plays it.

### Cookies (highly recommended)

Without cookies, YouTube live streams fall back to an anonymous client
that reliably gets HTTP 403'd on HLS segments after a few minutes —
this isn't an edge case, it's the default failure mode for live
stations. Uncomment and set this line in `stations.conf`:

```
cookies_browser=firefox
```

Any browser `yt-dlp --cookies-from-browser` supports works (`chrome`,
`brave`, `edge`, ...); use one you're actually logged into YouTube
with. `yt-play` picks it up automatically — no shell profile editing.
If `SYTMP_COOKIES_BROWSER` is set in the environment, that takes
priority over the config value (useful for a one-off override).

## Notes

- Live-stream video IDs can go stale if a channel restarts its broadcast
  under a new ID — if a station stops resolving, grab the current
  `/live` URL from the channel and update `stations.conf`.
- Regular (non-live) videos can occasionally be unplayable regardless of
  client/cookies (a YouTube-side PO-token requirement `yt-dlp` can't
  always satisfy) — this is an upstream `yt-dlp` limitation, not
  something `yt-play` can work around.
- Reconnect here is exit-and-respawn: if mpv crashes or the stream dies,
  it restarts. It does not poll mpv's internal playback clock to catch
  a stream that's still "running" but silently frozen (audio-output
  hang with no process exit) — a rarer edge case seen on some hardware
  paths. If you hit that, a property poll on `time-pos` via mpv's
  `--input-ipc-server` socket is the fix; not built in here to keep
  this a single-file tool with no polling loop running at rest.
