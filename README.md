# dotfiles

Personal configuration managed with [chezmoi](https://www.chezmoi.io).
Single source of truth for all my machines (home + work).

## Mental model

```
   git remote (GitHub)
        │  push / pull
        ▼
   SOURCE DIR   ~/.local/share/chezmoi   ← the "master copy", this repo
        │  chezmoi apply  (generates the files)
        ▼
   TARGET       $HOME                     ← the real files (niri, fish, ...)
```

- **Source dir** (`~/.local/share/chezmoi`): this git repo. The only thing you commit/push.
- **Target** (`$HOME`): the real files, *generated* by chezmoi with `apply`. They are copies, not symlinks — which is what makes templates possible.
- **Naming**: files in the source have names that chezmoi translates:
  - `dot_config/niri/...` → `~/.config/niri/...`
  - `dot_zshrc` → `~/.zshrc`
  - `.tmpl` suffix → the file is a template

## Setup on a new machine

```sh
sh -c "$(curl -fsLS get.chezmoi.io)"          # install chezmoi
chezmoi init --apply git@github.com:thegabriele97/dotfiles.git
```

Clones this repo and rebuilds the home directory in one shot.

## After `chezmoi apply` — new-machine checklist

`chezmoi apply` restores the *config files*, but some tools regenerate their own
files or need to be installed. Do this on every fresh machine (things I forget):

1. **Editor** — set nvim for chezmoi:
   ```sh
   mkdir -p ~/.config/chezmoi
   printf '[edit]\n    command = "nvim"\n' > ~/.config/chezmoi/chezmoi.toml
   ```
2. **fish plugins (fisher)** — `functions/`, `completions/`, most of `conf.d/`
   are NOT tracked; fisher regenerates them from `fish_plugins`:
   ```sh
   curl -sL https://raw.githubusercontent.com/jorgebucaran/fisher/main/functions/fisher.fish | source && fisher install jorgebucaran/fisher
   fisher update
   ```
3. **DankMaterialShell (DMS)** — generates the niri `dms/` files and the kitty
   `dank-*.conf` theme (both NOT tracked). Install/run DMS so those includes
   resolve. Without it niri and kitty still start, just without the dank theme.
4. **CLI deps used by my fish functions** — install so `cat`/`ff`/greeting work:
   `bat` (batcat), `fastfetch`, `eza`, `lolcat`, `fzf`.
5. **vim plugins** — TODO: document plugin-manager step once vim is set up.

## Daily workflow

Bring a file into chezmoi (the first time):

```sh
chezmoi add ~/.config/niri/gabos/irin.kdl
```

Edit a config:

```sh
chezmoi edit ~/.config/niri/gabos/binds.kdl   # opens the SOURCE
chezmoi apply                                  # regenerates the real file in ~
chezmoi diff                                   # dry-run: what would change
```

Sync between the two PCs:

```sh
# on the PC where you made changes:
chezmoi cd
git add -A && git commit -m "..." && git push
exit

# on the other PC:
chezmoi update      # = git pull + chezmoi apply
```

## Per-machine differences (templates)

Small differences (e.g. niri window rules that differ between home and work)
are handled with Go templates. The shared config stays pure KDL; only the
machine-specific bits live in a small templated file that niri includes.

How niri is wired:

- `gabos/window_rules.kdl` — rules shared by every machine (pure KDL).
- `gabos/window_rules.kdl` ends with `include optional=true "window_rules_local.kdl"`.
- `gabos/window_rules_local.kdl` is generated per-machine from the template
  `window_rules_local.kdl.tmpl`:

```kdl
{{- if eq .chezmoi.hostname "PC-LAGRECA" }}
// --- WORK (PC-LAGRECA) ---
window-rule { match app-id=r#"VirtualLCD"#; opacity 1.0; }
{{- else }}
// --- HOME ---
{{- end }}
```

`optional=true` means niri won't error if the local file is missing.
Never edit the generated `~/.config/.../window_rules_local.kdl` directly — edit
the template with `chezmoi edit` (chezmoi's `re-add` skips template files).

Handy variables: `.chezmoi.hostname`, `.chezmoi.os`, `.chezmoi.arch`.
See the available values with `chezmoi data`.

## Most-used commands

| Command | What it does |
|---|---|
| `chezmoi add <file>` | bring a file from ~ into the source |
| `chezmoi edit <file>` | edit a file's source |
| `chezmoi apply` | regenerate the real files in ~ from the source |
| `chezmoi diff` | show what would change (dry-run) |
| `chezmoi update` | git pull + apply (sync from the other PC) |
| `chezmoi cd` | enter the source to git commit/push |
| `chezmoi re-add` | update the source from files edited in ~ |
| `chezmoi data` | show the variables available in templates |

## What is NOT tracked (and how it's restored)

Anything an app regenerates or that is machine-specific stays out of chezmoi
(see `.chezmoiignore`). Restored by the new-machine checklist above.

| Not tracked | Owner / how it comes back |
|---|---|
| `~/.config/niri/dms/` | DankMaterialShell (dynamic theme, per-machine monitors) |
| `~/.config/kitty/dank-*.conf` | DankMaterialShell theme generator |
| `~/.config/fish/{functions,completions,conf.d}` (plugin files) | `fisher update` from `fish_plugins` |
| `~/.config/fish/fish_variables` | fish runtime state (universal variables) |
| `~/.config/chezmoi/chezmoi.toml` | chezmoi's own per-machine config |

Only my 3 hand-written fish functions (`cat`, `ff`, `fish_greeting`) are tracked
individually — never `chezmoi add ~/.config/fish/functions` as a whole dir.

## Notes

- `README.md` is in `.chezmoiignore`: it stays in the repo but is not copied to `~`.
