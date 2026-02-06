# Changelog

All notable changes to this project will be documented in this file, following [Semantic Versioning (SemVer)](https://semver.org/) standards.

## [3.0.1] - 2026-02-05
### Patch - Hidden Treasure Discovery 🗺️
Discovered helpful features and optimizations within Git, React, and shell tooling.

#### 🛠️ Git Discoveries
- **Local-only ignore patterns**: Learned to use `.git/info/exclude` for personal scratch files (created `_alex_workbench/` folder)
- **Reflog safety net**: Discovered `git reflog` can recover "lost" commits for ~90 days after hard resets or deleted branches
- **Configurable git aliases**: Created smart aliases with bash parameter expansion (`git undo`, `git undo-to`, `git history`) that default to safer options
- **Shell vs git aliases**: Understood when to use shell aliases (`.zshrc`) vs git aliases (`.gitconfig`) for portability

#### ⚛️ React Discoveries
- **Programmatic input events**: Learned to trigger React's `onChange` handlers by accessing native HTMLInputElement setters to bypass React's internal event tracker

#### 🐚 Shell Discoveries
- **Symlinks**: Learned to use symbolic links (`ln -s`) to access files from multiple locations without duplication

## [3.0.0] - Initial Release
### Added
- Created the progress tracker repository.
- Established the changelog system to document personal and professional growth. 