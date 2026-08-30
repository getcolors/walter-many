# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## What this repository is

Desired state for `walter-many`: one Vultr development machine with **seats** —
extra unix logins beside `ubuntu`, one person's isolated workspaces kept apart
by file permissions. OpenTofu state lives in Cloudflare R2. Nothing here is
source code: what is tracked is `colors.yml`, the installed launcher, the skill
package behind it, and the dev-environment files.

```text
colors.yml                        the desired state — the only file you normally edit
green                             the installed launcher (a COPY of the payload)
.agents/skills/package-walter-green   the installed skill package
.claude/skills/package-walter-green   a symlink into .agents/skills
.envrc                            secret-free; sources the gitignored .envrc.private
devenv.nix                        the toolchain
```

Everything else is generated (`.colors/`) or secret (`.envrc.private`).
`.gitignore` is `.*` with narrow negations, so check `git ls-files` rather than
assuming from the working tree.

## Commands

```sh
./green build              # render .colors/walter-many/ — contacts nothing
./green create --dry-run   # print the graph — touches nothing
./green create             # provision (device-flow code first, then unattended)
./green stop               # power off
./green start              # power on, and refresh the aliases
./green delete             # destroy (guarded — see below)
```

Toolchain comes from `devenv` via `direnv` — run `direnv allow` once. The
walter verbs need `tofu` and `ansible-playbook`, which devenv provides.

## The seats

`users: [jack, emma]` in `colors.yml` gives the machine three logins:

| Alias | Login | Role |
|---|---|---|
| `ssh walter-many` | ubuntu | trust root; the only sudoer; system-level work |
| `ssh walter-many-jack` | jack | seat: isolated workspace, no sudo |
| `ssh walter-many-emma` | emma | seat: isolated workspace, no sudo |

Every login gets the same environment, each in its own home: the nix profile,
fish, the Emacs configuration at `~/.config/neoemacs` (launch with
`emacs --init-directory ~/.config/neoemacs`), the dotfiles, the full
`~/code/getcolors` checkout tree, the seeded agent CLI logins, and the shared
atuin account. Seats are operated through ssh + zellij sessions; detached
zellij servers survive disconnect.

The isolation contract, stated plainly:

- **Never grant a seat sudo.** A sudoer can read every home; granting it
  deletes the feature. ubuntu is where system-level work happens.
- Seat homes are mode `0700`; seats cannot read each other's files. The
  primary `ubuntu` home may remain mode `0750`; seats are not members of its
  group and cannot read it.
- The boundary is filesystem and process, **not network or identity**: seats
  share localhost, `/tmp`, one GitHub token, one atuin history (directory
  filter recalls per workspace), and the seeded agent credentials.
- Agent CLI credentials are copies of the workstation's, one per home, so all
  homes share each CLI's refresh token. A rotation in one home can log
  another out — including this workstation. That is the accepted trade:
  `/login` there when it happens; the `force: false` seed guard means no
  later create clobbers the fresh session.

## State isolation

Shared R2 bucket, remote state keyed `<profile>/<stage>.tfstate`; this project
writes `walter-many/walter-compute.tfstate`.

**Never export `COLORS_PAR_PROFILE`.** Walter refuses to start when it is set —
an environment override could point it at another deployment's state. That is
the guard working; do not suggest a workaround.

## Gotchas

**The root `green` is a copy, not a symlink.** After `npx skills update -p`,
re-copy or the project keeps running the old pin:

```sh
npx skills update -p
cp .agents/skills/package-walter-green/green green
```

**Vultr bills a stopped instance.** `stop` halts the machine and keeps the
aliases honest, but unlike OCI the compute reservation still bills. `delete`
is the way to stop paying entirely, and it takes the disk with it.

**Fill in `vultr-instance-id` after the first create**, and again after every
recreate. It is what lets `stop`/`start` work when the R2 backend is
unreachable. Read it back with:

```sh
cd .colors/walter-many/walter-compute
AWS_ACCESS_KEY_ID="$COLORS_PAR_R2_ACCESS_KEY_ID" \
AWS_SECRET_ACCESS_KEY="$COLORS_PAR_R2_SECRET_ACCESS_KEY" \
  tofu output -raw instance_id
```

(A bare `tofu` falls through the AWS SDK's default chain to unrelated
credentials and fails with a confusing key-length error.)

**A recreate brings a new host key and address.** The managed blocks are
rewritten, but `~/.ssh/known_hosts` is not touched — the first
`ssh walter-many` afterwards asks to accept an unknown host. All aliases share
one `HostName`, so accepting once covers the seats too.

**`delete` takes the disk with it.** The guard is on by default; lift it for
one intentional run with `COLORS_PAR_COMPUTE_PREVENT_DESTROY=false`.

## Provenance

The launcher is a real install: `skills-lock.json` records what
`npx skills add getcolors/walter` computed. `./green` resolves
`io.github.getcolors/walter` at the stamped commit into `~/.gitlibs` on first
run — no checkout, no `WALTER_LIB_ROOT`, no install step.

## Git

Do not commit or push unless explicitly asked.
