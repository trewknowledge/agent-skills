# Agent Skills

A collection of skills for AI coding agents. Skills are packaged instructions and scripts that extend agent capabilities.

## Available Skills

<!-- SKILLS:START -->
| Skill | Description |
|-------|-------------|
| [wordpress-vip](skills/wordpress-vip/) | When the user is working a WordPress VIP project |
| [project-documentation](skills/project-documentation/) | When the user wants to document a feature for non-developers and AI Agents |
| [release](skills/release/) | When the user wants to cut a release and document the changes |
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

Once installed, it should use the skills when they are relevant. You may want to provide context directly when you engage a new agent, such as whether you are working on a WordPress VIP project.

You can also invoke skills directly:

```
/wordpress-vip
```

## How to Create a new Skill
This repository's AGENTS.md file contains instructions on creating a new skill so that an agent follows the same type of language and structure.

*Cursor:* Cursor has a /create-skill command available. It will create it under a .cursor folder. You then move it to the skills folder we already have.
*Claude Code:* There's a skill by Anthropic called [Skill Creator](https://github.com/anthropics/skills/tree/main/skills/skill-creator) included in this repository. Move it to the main skills folder when done.

## Contributing

Found a way to improve a skill? Have a new skill to suggest? PRs and issues welcome!
