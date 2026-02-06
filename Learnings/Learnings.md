# Learnings Changelog

A running log of useful things I've learned while building UI.

---

## 2026-02-05

### 🛠️ Git: Local-only ignore patterns with `.git/info/exclude` ✅ Shared
- **Problem**: Personal research/scratch files cluttering `git status`
- **Solution**: Use `.git/info/exclude` instead of `.gitignore` for local-only patterns
- **Why it's better**: No PR needed, doesn't affect team, file never shows as modified
- **Setup**: Created `_alex_workbench/` folder and added it to `.git/info/exclude`
- **Related**: Also learned about `~/.gitignore_global` for cross-repo patterns

---

### 🛠️ Git: `reflog` as the ultimate safety net
- Tracks every HEAD movement for ~90 days
- Can recover "lost" commits after hard resets, deleted branches, botched rebases
- Commands: `git reflog`, `git reset --hard HEAD@{n}`
- Local only, doesn't sync with remote

---

### ⚛️ React: Programmatically simulating input events ✅ Shared
- **Problem**: Setting `input.value` and dispatching events doesn't trigger React's `onChange` handlers
- **Why**: React tracks input values internally and suppresses duplicate events
- **Solution**: Use the native HTMLInputElement setter to bypass React's tracker
- **Code**:
```javascript
const nativeSetter = Object.getOwnPropertyDescriptor(
  window.HTMLInputElement.prototype,
  'value'
)?.set;
nativeSetter?.call(input, 'x');
input.dispatchEvent(new Event('input', { bubbles: true }));
```
- **Use cases**: Testing, automation, complex interactions like Direct Edit-on-Focus

---

### 🛠️ Git: Configurable defaults in git aliases with bash parameter expansion 
- **Problem**: Wanted git undo to default to `--soft` (safer) but allow `--hard` override
- **Solution**: Use `${parameter:-default}` syntax in git alias functions
- **Code**:
```bash
[alias]
    undo = "!f() { local type=${1:---soft}; git reset $type HEAD@{1}; }; f"
    undo-to = "!f() { local type=${2:---soft}; git reset $type HEAD@{$1}; }; f"
    history = reflog --pretty=format:\"%C(yellow)%h%Creset %C(cyan)%gd%Creset %gs\" -10
```
- **How it works**: `${1:---soft}` means "use first argument, or default to `--soft` if not provided"
- **Real usage**:
  - `git undo` → defaults to soft (keeps changes staged)
  - `git undo --hard` → override to discard changes
  - `git undo-to 3` → jump to HEAD@{3} with soft reset
  - `git undo-to 3 --hard` → jump with hard reset
  - `git history` → see last 10 reflog entries with pretty colors
- **Why this matters**: Safer defaults (soft) prevent accidental data loss, while preserving flexibility
- **Pattern to reuse**: `"!f() { local var=${position:-default}; command $var; }; f"`

---

### 🛠️ Git: Shell aliases vs git aliases - When to use which
- **Shell aliases** (`.zshrc`): Direct commands like `gp='git push'`
- **Git aliases** (`.gitconfig`): Git subcommands like `git undo`
- **Key differences**:
  - Shell aliases: Fast to type, shell-specific, may not work in IDEs
  - Git aliases: Portable, work in any git context (VS Code, Tower, etc.)
- **When to use shell**: Quick shortcuts you type frequently (`gp`, `gs`)
- **When to use git**: Portable workflows, complex operations, IDE compatibility
- **Real setup**: Shell for speed (`gp`), git for workflows (`git undo`, `git history`)

---

### 🛠️ Symlinks: Access files from multiple locations without duplication
- **Problem**: Need to access the same file from multiple locations without copying it
- **What they are**: Pointers/shortcuts to files or folders in another location
- **How they work**: Like a portal - the file appears to be in multiple places, but physically exists in only one
- **Create**: `ln -s /path/to/real/file /path/to/symlink`
- **Remove**: `rm symlink` (safe - doesn't delete the target)
- **Real usage**:
  - Shared learnings across repos: `ui/_alex_workbench/Learnings.md → ~/personal-repo/Learnings.md`
  - Local package development: `node_modules/my-lib → ~/projects/my-lib`
  - Configuration management: `~/.vimrc → ~/dotfiles/.vimrc`
  - Version switching: `/usr/bin/python → /usr/bin/python3.11`
- **Why they matter**: 
  - Avoid duplication (update once, affects all)
  - Safer (files survive repo deletion)
  - Used heavily by npm, Docker, git, Python venv, Homebrew
- **When NOT to use**: Across filesystems, Windows, in committed git repos (paths won't exist for others)

---

## Template for future entries

### 🛠️ [Topic]: [One-line summary] [✅ Shared if in PAINT_THE_DOM]
- **Problem**: What was I trying to solve?
- **Solution**: What did I learn/discover?
- **Why it matters**: Why is this useful?
- **Commands/Code**: Key snippets to remember

---

**Emoji categories:**
- 🛠️ Tooling (Git, CI/CD, Build tools)
- ⚛️ React
- 🐚 Shell/Bash
- 🎨 CSS/Styling
- 📦 Package Management
- 🔍 Debugging
- 🧪 Testing

**Formatting:**
- Use horizontal rules (`---`) between entries for visual separation
- Add `✅ Shared` marker if the learning is already in PAINT_THE_DOM_SLACK.md
