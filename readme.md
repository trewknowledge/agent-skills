# Agent Skills

A collection of skills for AI coding agents. Skills are packaged instructions and scripts that extend agent capabilities.

## Available Skills

<!-- SKILLS:START -->
| Skill | Description |
|-------|-------------|
| [wordpress-vip](skills/wordpress-vip/) | When the user is working a WordPress VIP project |
| [project-documentation](skills/project-documentat/) | When the user wants to document a feature for non-developers and AI Agents |
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