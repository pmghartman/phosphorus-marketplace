# Phosphorus plugin

The Claude skill for a Phosphorus mesh. Published from `plugin/` in the phosphorus repository on each release; edits here are replaced.

The marketplace also holds `bismuth`, the skill for the Bismuth language, published to `plugins/bismuth` by each bic release from ryan/bismuth (`tools/claude-skill/bismuth`). Install it with `/plugin install bismuth@phosphorus` in Claude Code, or as Bismuth from the same marketplace in claude.ai.

1. Add the marketplace and install the plugin:
   - Claude Code: `/plugin marketplace add https://github.com/pmghartman/phosphorus-plugin`, then `/plugin install phosphorus@phosphorus`. Turn on auto-update for the marketplace under `/plugin`, Marketplaces.
   - claude.ai, Claude Desktop or Cowork: Customize, Plugins, Add marketplace: `https://github.com/pmghartman/phosphorus-plugin`, then install Phosphorus.
2. Add your mesh's connector, `https://<your mesh domain>/api/mcp` (your mesh desktop's Agents app, Setup, shows it):
   - Claude Code: `claude mcp add --transport http --scope user phosphorus https://<your mesh domain>/api/mcp`
   - claude.ai, Claude Desktop or Cowork: Customize, Connectors, Add custom connector.

The connector signs in with your mesh user and password. What it may do follows your role.
