# Django Guidelines & Best Practice

Shared coding guidelines for Django projects, maintained in one place and
consumed by other repos as a **git submodule**.

Editing this file here propagates to every project on the next
`git submodule update --remote`.

---

## What's in here

| File | Purpose |
|------|---------|
| `guidelines.md` | Coding standards, conventions, and best-practice rules for Django projects |

---

## Using this in a project

### Add as a submodule (first time)

```bash
git submodule add https://github.com/andytwoods/DjangoGuidelinesBestPractice.git .junie
git commit -m "Add shared guidelines submodule"
```

This places `guidelines.md` at `.junie/guidelines.md`, where tools like
[JetBrains Junie](https://www.jetbrains.com/junie/) and
[Claude Code](https://claude.ai/code) pick it up automatically.

### Pull the latest guidelines into a project

```bash
git submodule update --remote .junie
git commit -m "Update shared guidelines"
```

### Update all projects at once

Add this to your shell profile (`~/.zshrc` or `~/.bashrc`):

```bash
update-guidelines() {
  for dir in ~/PycharmProjects/*/; do
    if [ -f "$dir/.gitmodules" ]; then
      echo "Updating $dir..."
      git -C "$dir" submodule update --remote .junie && \
      git -C "$dir" commit -m "Update shared guidelines" .junie
    fi
  done
}
```

Then run `update-guidelines` from anywhere.

---

## Cloning a project that uses this submodule

Submodules are not fetched by default. Use either:

```bash
# Clone and fetch submodules in one step
git clone --recurse-submodules <project-url>

# Or, if already cloned
git submodule update --init
```

---

## Contributing / editing guidelines

1. `cd .junie/` inside any project that uses this submodule
2. Edit `guidelines.md`
3. Commit and push — the change lands in this central repo immediately
4. Other projects pick it up on their next `git submodule update --remote`

---

## Tooling compatibility

`guidelines.md` is read automatically by:

- **Claude Code** — place the submodule at `.junie/` and reference it in `CLAUDE.md`:
  ```
  Before making any code changes, read and follow: `.junie/guidelines.md`
  ```
- **JetBrains Junie** — reads `.junie/guidelines.md` natively
