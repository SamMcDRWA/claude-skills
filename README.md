# claude-skills

Personal Claude Code plugin marketplace.

## Plugins

### review-loop

Iterative code review skill: orient → review → present findings → fix → self-review → repeat. Covers security, logic bugs, implementation gaps, second-order effects, loose ends, and AI-slop / code bloat. Uses a delete-first principle and Ousterhout's deep-modules test for minimalism.

## Install

Add this marketplace to your Claude Code, then install the plugin:

```bash
/plugin marketplace add SamMcDRWA/claude-skills
/plugin install review-loop@claude-skills
```

Or, edit `~/.claude/settings.json` directly:

```json
{
  "extraKnownMarketplaces": {
    "claude-skills": {
      "source": {
        "source": "git",
        "url": "https://github.com/SamMcDRWA/claude-skills.git"
      }
    }
  },
  "enabledPlugins": {
    "review-loop@claude-skills": true
  }
}
```

Restart Claude Code, then trigger with phrases like:
- "do a review pass on this project"
- "/review-loop"
- "audit for bugs and AI slop"
