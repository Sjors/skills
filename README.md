# Skills

Some useful skills for your favorite agent.

They're somewhat tailored to Bitcoin Core adjacent projects and a macOS
development environment. Limited pull requests to make them Linux friendly are
welcome.

## Suggested layout

Keep a normal folder for your skills, with separate repositories for published
and unpublished work:

```text
~/skills/
├── .system/             # Codex-managed skills, if installed here
├── public/              # Clone of this repository
├── local/               # Your unpublished skills; optionally a private Git repo
├── rebase -> public/rebase
└── my-skill -> local/my-skill
```

The top-level skill links let Claude Code discover each skill directly and keep
paths stable when a skill moves between repositories. Codex also supports
symlinked skills and grouped subdirectories. Keep bundled `.system` files under
Codex's control, outside both repositories.

For a new installation:

```sh
mkdir -p "$HOME/skills/local"
git clone https://github.com/Sjors/skills.git "$HOME/skills/public"
ln -s public/rebase "$HOME/skills/rebase"
```

Review the skills and link the ones you want. After creating your own skill,
you can install it the same way:

```sh
ln -s local/my-skill "$HOME/skills/my-skill"
```

## Connect your agent

If the destination does not already exist, point the agent's skills folder at
your shared folder:

```sh
mkdir -p "$HOME/.codex" "$HOME/.claude"
ln -s "$HOME/skills" "$HOME/.codex/skills"
ln -s "$HOME/skills" "$HOME/.claude/skills"
```

If you already have a skills directory, leave it and its bundled skills in
place and add individual links inside it instead:

```sh
ln -s "$HOME/skills/public/rebase" "$HOME/.codex/skills/rebase"
ln -s "$HOME/skills/public/rebase" "$HOME/.claude/skills/rebase"
```

See the official [Codex skill documentation](https://learn.chatgpt.com/docs/build-skills)
and [Claude Code skill documentation](https://code.claude.com/docs/en/skills#choose-where-skills-load)
for discovery locations and symlink support.

## Keep unpublished skills separate

Use `public/` for commits intended for this repository. Keep your own skills in
`local/`, with separate Git history and your own backup or SSH sync remote.
Keep documentation links relative to their location in this repository;
cloning it into `public/` does not change its paths on GitHub. Some workflows
refer to optional unpublished companion skills. Use ignored checkout-local
symlinks to make your own equivalents available without changing those links.

To version unpublished work without publishing it:

```sh
git -C "$HOME/skills/local" init
```

No submodule is needed: the shared parent is not a repository, and each sibling
can be committed and synced independently.
