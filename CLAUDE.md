# Installing

yabai is installed only by the dotfiles script, run manually by the user:
`~/.dotfiles/scripts/update_yabai_skhd.sh`. It clones `origin/fix-arc-fs-new`
from GitHub, runs `make install` + `make sign`, installs to `/usr/local/bin`,
rewrites the sudoers checksum for `--load-sa`, and restarts the service.

- Never install by hand: no `sudo install`/`cp`/`mv` into `/usr/local/bin`, no
  editing `/private/etc/sudoers.d/yabai`, no `yabai --restart-service` as a
  substitute for the script.
- A change reaches the machine only after it is committed and pushed to
  `origin/fix-arc-fs-new`; the script builds from the remote, not this checkout.
- Local `make install` in this checkout only produces `bin/yabai` for inspection.
