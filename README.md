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
are handled with Go templates: rename the file with a `.tmpl` suffix and
branch on the hostname. One shared file, no forking.

```kdl
// gabos/window_rules.kdl.tmpl
window-rule {
    match app-id="slack"
    {{ if eq .chezmoi.hostname "PC-LAGRECA" }}
    open-on-workspace "work"
    {{ end }}
}
```

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

## Notes

- `README.md` is in `.chezmoiignore`: it stays in the repo but is not copied to `~`.
