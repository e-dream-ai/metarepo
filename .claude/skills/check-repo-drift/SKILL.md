---
name: check-repo-drift
description: Survey how far each metarepo sibling checkout is behind/ahead of its upstream, and read deployed code from origin without touching working trees. Use before trusting local source for questions about deployed behavior.
---

# Check repo drift

Survey drift across every repo (fetch is read-only and safe to run any time):

```bash
cd metarepo
for d in */; do
    [ -d "$d/.git" ] || [ -f "$d/.git" ] || continue
    git -C "$d" fetch --quiet --all --prune 2>/dev/null
done

printf '%-24s %-7s %-6s %s\n' REPO BEHIND AHEAD BRANCH
for d in */; do
    [ -d "$d/.git" ] || [ -f "$d/.git" ] || continue
    b=$(git -C "$d" rev-parse --abbrev-ref HEAD 2>/dev/null)
    if up=$(git -C "$d" rev-parse --abbrev-ref '@{u}' 2>/dev/null); then
        set -- $(git -C "$d" rev-list --left-right --count "$up"...HEAD 2>/dev/null)
        printf '%-24s %-7s %-6s %s -> %s\n' "${d%/}" "$1" "$2" "$b" "$up"
    else
        printf '%-24s %-7s %-6s %s (no upstream)\n' "${d%/}" "?" "?" "$b"
    fi
done
```

Compare against `@{u}` (the branch's own upstream), not `origin/main`. Several
repos sit on `stage` or a feature branch, where a behind-count against `main` is
meaningless noise. `AHEAD > 0` means unpushed local commits — look before you
pull.

When an answer depends on what is actually deployed, `git fetch` and read
`origin/main` directly (`git show origin/main:path/to/file`, `git grep -n pat
origin/main`) rather than the working tree. That inspects the remote state
without touching a checkout that may hold someone's in-progress work.
