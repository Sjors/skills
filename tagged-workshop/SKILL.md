---
name: tagged-workshop
description: Create and maintain Git-tagged code workshops/tutorials with step tags, exercise/solution tags, GitHub navigation links, CI over tags, and history rewrites or rebases when intermediate steps change. Use when a user asks to split a tutorial into commits/tags, retag step.* refs, update a workshop branch after editing the final result, or maintain a master/landing-page branch plus a workshop branch.
---

# Tagged Workshop

## Model

Use this skill for repositories where a tutorial is encoded as Git history:

- A public landing branch such as `master` contains the landing page, setup, and CI.
- A `workshop` branch contains a linear sequence of tutorial commits.
- Tags such as `step.1`, `step.2`, `step.4.solution` point at specific commits. In some workshops `step.1` is the landing/base commit on `master`, with later tags on `workshop`.
- README navigation is forward-only and links to GitHub tag pages, not local anchors.

Prefer preserving public branch history. Force-push the `workshop` branch and moved tags when steps are rewritten. Do not force-push `master` after publication unless the user explicitly asks; for Step 1/landing-page changes, make a new commit on `master`, then rebase/rebuild `workshop` on top of the updated base and retag.

## Navigation Format

Use GitHub tag URLs:

```md
Use `git checkout step.2` to move to [step 2](https://github.com/OWNER/REPO/tree/step.2).
Use `git checkout step.4.solution` to see the [Step 4 solution](https://github.com/OWNER/REPO/tree/step.4.solution).
```

Do not add back navigation unless the user asks.

## Standard Workflow

1. Inspect state:

```sh
git status --short --branch
git branch -vv
git tag --list 'step.*' --sort=version:refname
git log --oneline --decorate --graph --all --max-count=20
```

2. Create a safety branch before rewriting workshop history:

```sh
git branch backup/workshop-$(date +%Y%m%d%H%M%S) workshop
```

3. Identify the earliest affected step.

If the user edited the final result first, inspect the diff and apply only the relevant pieces to the earliest commit where they should first appear. Later commits should inherit the change through replay/rebase.

4. Rewrite the `workshop` branch.

For small changes, use interactive rebase from the parent of the earliest affected commit. For larger changes or many retags, rebuilding is often clearer:

```sh
git switch workshop
git reset --hard <base-commit>
git cherry-pick <old-step-1-commit>
# apply/edit/amend as needed
git cherry-pick <old-step-2-commit>
# repeat
```

5. Retag every moved step.

Use `/Users/dev/utils/tagged_workshop_retag.py` when the branch is linear and each tag maps to one commit in order:

```sh
/Users/dev/utils/tagged_workshop_retag.py \
  --branch workshop \
  --base <base-commit-or-ref> \
  --tags step.1,step.2,step.3,step.4,step.4.solution,step.5,step.5.solution
```

If the first step is the landing/base commit on `master`, tag the base separately:

```sh
/Users/dev/utils/tagged_workshop_retag.py \
  --branch workshop \
  --base master \
  --base-tag step.1 \
  --tags step.2,step.3,step.4,step.4.solution,step.5,step.5.solution \
  --ci-workflow .github/workflows/ci.yml
```

If commit order and tag order do not match exactly, retag manually with `git tag -f <tag> <commit>`.

6. Validate all tags locally.

Run the same checks CI will run. It is fine to mark early steps as exceptions when they intentionally have no Rust/project files yet. If CI hardcodes tag names as separate steps, the local validation order and CI step list must be updated whenever tags are added, removed, or renamed.

```sh
current_branch=$(git branch --show-current)
for tag in $(git tag --list 'step.*' --sort=version:refname); do
  git checkout --force "$tag"
  case "$tag" in
    step.1|step.2) echo "$tag has no project yet; skipping" ;;
    *) cargo fmt --check && cargo check --locked ;;
  esac
done
git checkout --force "$current_branch"
```

7. Push with lease.

```sh
git push --force-with-lease origin workshop \
  +refs/tags/step.1:refs/tags/step.1 \
  +refs/tags/step.2:refs/tags/step.2
```

Include every tag that moved. Push new tags normally or with the same explicit refspec.

## Updating Step 1 / Landing Page

Step 1 often also appears on `master`. After the project is public:

1. Switch to `master`.
2. Make the landing-page change as a new commit.
3. Push `master` normally.
4. Rebase or rebuild `workshop` on the updated master/base.
5. Retag affected steps.
6. Force-push only `workshop` and moved tags.

This avoids force-pushing `master` while keeping the workshop history coherent.

## CI Pattern

Add CI on the landing branch that fetches all tags and checks every `step.*` tag. Install any system dependencies required by later steps. Example checks:

- `cargo fmt --check`
- `cargo check --locked`

Skip intentional non-code steps explicitly by tag name rather than letting them fail.

For the current Bitcoin Core IPC workshop layout:

- `master` contains CI, setup text, deterministic fixtures under `test/fixtures`, and may be tagged as `step.1`.
- `workshop` is rebased onto `master` for landing/setup changes, then later step tags are moved.
- CI uses one runner with separate `Check step.*` steps, not a matrix. Keep those steps in the same order as the tag list.
- CI starts Bitcoin Core v31 from the release binary, loads fixture blocks, and runs later steps with `--ci` so it does not mine real blocks.
- Rust compiler artifacts are cached with `sccache`; disable the setup-rust-toolchain cache when the landing branch has no `Cargo.toml`.
- When tags change, update CI in the same change and verify the workflow mentions every moved/new `step.*` tag.

## Failure Rules

- If a rewrite goes wrong, stop and use the backup branch rather than improvising.
- Never delete or move public tags without force-pushing the corrected tags.
- Never assume GitHub tag links work until the tags have been pushed.
- After a force-push, verify with `git ls-remote --heads --tags origin`.
