# Phosphorus plugin

The Claude plugin for a Phosphorus mesh. Published from `plugin/` in the phosphorus repository on each release; edits here are replaced.

- Claude Code: `/plugin marketplace add https://github.com/pmghartman/phosphorus-plugin`, then `/plugin install phosphorus@phosphorus` and enter your mesh's domain when asked (`/plugin configure phosphorus@phosphorus` changes it). Turn on auto-update for the marketplace under `/plugin`, Marketplaces.
- claude.ai, Claude Desktop or Cowork: Customize, Plugins, Add marketplace: `https://github.com/pmghartman/phosphorus-plugin`. These apps don't ask for the domain, so also add the connector under Customize, Connectors, Add custom connector, with the URL `https://<your mesh domain>/api/mcp`.

The connector signs in with your mesh user and password. What it may do follows your role.
