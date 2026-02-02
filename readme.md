# Trew Knowledge Skills

A collection of AI agent skills focused on marketing tasks. Built for technical marketers and founders who want Claude Code (or similar AI coding assistants) to help with conversion optimization, copywriting, SEO, analytics, and growth engineering.

## What are Skills?

Skills are markdown files that give AI agents specialized knowledge and workflows for specific tasks. When you add these to your project, Claude Code can recognize when you're working on a marketing task and apply the right frameworks and best practices.

## Available Skills

<!-- SKILLS:START -->
| Skill | Description |
|-------|-------------|
| [wordpress-vip](skills/wordpress-vip/) | When the user is working a WordPress VIP project |
<!-- SKILLS:END -->

## Installation

### Option 1: CLI Install (Recommended)

Use [npx skills](https://github.com/vercel-labs/skills) to install skills directly:

```bash
# Install all skills
npx skills add trewknowledge/agent-skills

# Install specific skills
npx skills add trewknowledge/agent-skills --skill wordpress-vip

# List available skills
npx skills add trewknowledge/agent-skills --list
```

This automatically installs to your `.claude/skills/` directory.

### Option 2: Clone and Copy

Clone the entire repo and copy the skills folder:

```bash
git clone https://github.com/trewknowledge/agent-skills.git
cp -r agent-skills/skills/* .claude/skills/
```
## Usage

Once installed, it should use the skills when they are relevant. You may want to provide context directly at the beginning such as if you are working on a WP VIP project or not.

You can also invoke skills directly:

```
/wordpress-vip
```

## Contributing

Found a way to improve a skill? Have a new skill to suggest? PRs and issues welcome!