# walter-many

Desired state for one Vultr development machine with **seats**: extra unix
logins beside `ubuntu` — one person's isolated workspaces, kept apart by file
permissions — provisioned by [walter](https://github.com/getcolors/walter).

```sh
./green build              # render .colors/walter-many/ — contacts nothing
./green create --dry-run   # print the graph — touches nothing
./green create             # provision (one device-flow code, then unattended)
./green stop | start       # power cycle
./green delete             # destroy (guarded)
```

After a create:

```sh
ssh walter-many            # ubuntu — the trust root, the only sudoer
ssh walter-many-jack       # a seat: private home, no sudo, full toolchain
ssh walter-many-emma       # another seat
```

Every login gets the same environment in its own home — nix profile, fish,
Emacs configuration, dotfiles, the org checkouts, agent CLI logins, one shared
atuin history. Seats hold no sudo: the isolation is the filesystem, and a
sudoer could read every home.

Desired state is `colors.yml`. Credentials never live there — they arrive as
`COLORS_PAR_*` environment variables from the gitignored `.envrc.private`.
See `CLAUDE.md` for the seat contract and the operational gotchas.
