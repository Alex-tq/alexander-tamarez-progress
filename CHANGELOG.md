# Changelog

All notable changes to this project will be documented in this file, following [Semantic Versioning (SemVer)](https://semver.org/) standards.

## [4.0.2] - 2026-03-30
### Patch - Hidden Treasure Discovery 🗺️
Deep dive into build tooling and package management concepts.

#### 🛠️ Build Tooling Discoveries
- **Tree-shaking fundamentals**: Learned what tree-shaking is and why it only works with ESM (not CJS). CJS allows dynamic `require()` calls at runtime (inside conditionals, loops, functions), forcing bundlers to keep all code. ESM `import` statements must be static and top-level, enabling safe dead code elimination.

#### 📦 Package Management Discoveries
- **Package exports and compilation**: Discovered how the `exports` field in `package.json` routes consumers to the right files (ESM, CJS, types). Learned that packages shipping raw TypeScript source break this contract, forcing every consumer to configure `transformIgnorePatterns`, path aliases, and loader exceptions manually.

## [4.0.1] - 2026-02-05
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

## [4.0.0] - Initial Release
### Starting Point - Why Version 4?

This version represents my baseline knowledge when establishing this tracking system. The version number reflects four distinct "seas" I've discovered:

- **v1.0.0**: React/TypeScript/Frontend Stack - Mastered modern frontend development (React, TypeScript, CSS, component architecture, accessibility, responsive design)
- **v2.0.0**: Testing & Professional Practices - Became proficient in testing as a craft and professional workflows (Cypress, Storybook, Git workflows, debugging, performance profiling)
- **v3.0.0**: Technical Leadership - Achieved senior/lead-level impact (mentoring, project leadership, cross-functional collaboration, technical communication, interviewing)
- **v4.0.0**: AI-Assisted Development - Adopted AI-powered development paradigm (Cursor, Claude, ChatGPT with prompt engineering)

### Current Skills by Category

#### ⚛️ React/TypeScript
- TypeScript with React
- React component architecture (class and functional components)
- Form handling and validation
- State management patterns
- Props and type safety
- Custom hooks
- Redux (state management)
- React Router (client-side routing)

#### 📦 Component Library Development
- Building reusable component libraries
- Form components and editors
- Custom-built table components from scratch
- Modal/Dialog patterns
- Component composition
- Design systems (component guidelines, design tokens, documentation patterns)

#### 📚 Third Party Libraries
- MaterialUI
- Radix UI
- react-grid-layout & ag-grid (grid layouts)
- Decimal.js (precision math)
- Moment.js (date handling)

#### ♿ Accessibility
- ARIA attributes and proper implementation
- Accessible form patterns
- Screen reader compatibility
- Keyboard navigation

#### 🛠️ Development Workflow
- Component documentation (Storybook)
- Figma (inspecting designs, extracting specs)
- Conditional feature rollouts
- Agile methodologies

#### 🧪 Testing
- Testing frameworks (Cypress E2E/Component/Snapshot, Bun, UI testing libraries)

#### 🔍 Debugging & Performance
- Browser DevTools
- Performance profiling

#### 🎨 CSS/Styling
- Modern CSS (Grid, Flexbox, Variables, Container Queries), SCSS
- Responsive and mobile-first design

#### 🛠️ Git Workflow
- Git-based development (branching, PRs, remote repositories, code review processes)

#### 👥 Technical Leadership
- Project leadership (planning, task breakdown, coordination, timelines)
- Mentoring and code review culture (pair programming, knowledge sharing, giving/receiving constructive feedback)
- Cross-functional collaboration (working with designers, product managers, QA, backend teams)
- Time management and prioritization (balancing multiple tasks, estimating work, managing interruptions, saying no to scope creep)
- Technical interviewing

#### 🤖 AI-Assisted Development
- AI coding assistants (Cursor, Claude, ChatGPT)
- Prompt engineering for development tasks

#### 📝 Documentation & Technical Communication
- Developer-facing content (guides, documentation, code examples)
- Strategic communication (RFCs, architecture docs, stakeholder presentations)

#### 🎓 Certifications
- freeCodeCamp: Responsive Web Design Certification
- freeCodeCamp: JavaScript Algorithms and Data Structures Certification
- freeCodeCamp: Front End Development Libraries Certification

### Meta
- Created the progress tracker repository
- Established the changelog system to document personal and professional growth 