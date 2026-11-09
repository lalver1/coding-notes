# Rewriting history

## Undoing commits

## Editing commits

## Fixing up commits

## Rebasing during a branching workflow

In a branching workflow, it's common to concurrently work on 2 different units of work, say in branches `A` and `B` (where `B` depends on code being introduced in `A`) but `A` will not be immediately merged into `main`. This creates a little bit of overhead when we update branch `A` and consequently need to update branch `B`. The situations outlined below are very common.

### 1. Simple Rebase (Branch `A` Has More Commits)

Use this when `B` was branched off `A`, you committed work to `B`, and then added more commits to `A`.

#### Git Graph

```text
Before:
A: --- A1 --- A2 (HEAD)
        \
B:       B1 (HEAD)

After:
A: --- A1 --- A2
               \
B:              B1' (HEAD)

```

#### Commands

```bash
# 1. Switch to branch B
git switch B

# 2. Rebase B onto the tip of A
git rebase A
```

### 2. Rebase `A` onto `main`, then rebase `B` onto updated `A` (`B` branched from tip of old `A`)

Use this when `B` branched off the _tip_ of `A` (`A2`), `main` moved forward, and you rebase `A` onto `main`. Because `A`'s commit hashes change during the rebase, you must prune old `A` history when updating `B`.

#### Git Graph

```text
1. Initial State:
main: --- M1 --- M2 (latest)
           \
A:          A1 --- A2
                     \
B:                    B1

2. After rebasing A onto main (git switch A && git rebase main):
main: --- M1 --- M2 --- A1' --- A2' (A)
           \
(old A):    A1 --- A2
                     \
B:                    B1 (B)

3. Final State after rebasing B onto new A:
main: --- M1 --- M2 --- A1' --- A2' (A)
                                   \
B:                                  B1' (B)

```

#### Commands

```bash
# Step 1: Rebase A onto the updated main branch
git switch A
git rebase main

# Step 2: Rebase B onto the new A
git switch B

# Option A: Using the commit hash of old A2 directly (available in git graph since it is still "part of branch B")
git rebase --onto A <commit-hash-of-old-A2>

# Option B: Using reflog (if run immediately after step 1, A@{1} is old A2)
git rebase --onto A A@{1}
```

### 3. Branch `B` branched from an older commit of `A` (`switch`-first approach)

Use this when `B` branched off an intermediate commit `A1` (which was the tip at that time), you later added `A2` to branch `A`, and then rebased `A` onto `main`.

#### Git Graph

```text
1. Initial State:
main: --- M1 --- M2 (latest)
           \
A:          A1 --- A2
             \
B:            B1

2. After rebasing A onto main (git switch A && git rebase main):
main: --- M1 --- M2 --- A1' --- A2' (A)
           \
(old A):    A1 --- A2
             \
B:            B1 (B)

3. Final State after rebasing B onto new A:
main: --- M1 --- M2 --- A1' --- A2' (A)
                                   \
B:                                  B1' (B)

```

#### Commands

```bash
# Step 1: Rebase A onto main
git switch A
git rebase main

# Step 2: Switch to branch B
git switch B

# Step 3: Find the commit hash of old A1 (the commit right before B's first commit)
git log --oneline

# Step 4: Replay only B's unique commits onto the tip of updated A
# Syntax: git rebase --onto <new-parent> <cut-off-point>
git rebase --onto A <commit-hash-of-old-A1>
```

#### Rule of thumb for `--onto`

When rewriting upstream history, `git rebase --onto <destination> <last-commit-to-discard>` lets you extract commits without dragging along obsolete base commits.
