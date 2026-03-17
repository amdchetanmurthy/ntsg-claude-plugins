# NTSG Claude Plugins

A collection of Claude Code plugins for AMD AI infrastructure and related tooling.

> **⚠️ Important:** Make sure you trust a plugin before installing, updating, or using it. Review each plugin's documentation and source code before use.

## Plugins

### AMD AI Infra Plugin

The AMD AI Infra plugin provides comprehensive tooling for managing and debugging AMD GPU infrastructure, including:

- ROCm diagnostics and management
- Scale-out networking diagnostics
- GPU topology visualization
- Knowledge base integration for troubleshooting
- Lab environment utilities

**Status:** In Development

See [`plugins/amd-ai-infra/`](plugins/amd-ai-infra/) for detailed documentation.

## Structure

```
ntsg-claude-plugins/
├── plugins/
│   └── amd-ai-infra/     # AMD AI Infrastructure plugin
├── .claude-plugin/
│   └── marketplace.json  # Marketplace metadata
└── README.md
```

## Future Plugins

This repository is structured to host additional plugins as needed. Each plugin should be placed in the `plugins/` directory with its own self-contained structure.

## Installation

### Local Development

1. Clone this repository:
   ```bash
   git clone <repository-url>
   cd ntsg-claude-plugins
   ```

2. Install individual plugins by symlinking to your Claude Code plugins directory:
   ```bash
   ln -s $(pwd)/plugins/amd-ai-infra ~/.claude/plugins/amd-ai-infra
   ```

3. Restart Claude Code or run `/plugin reload`

### From Marketplace

When published, plugins can be installed via:
```
/plugin install {plugin-name}@ntsg-claude-plugins
```

command line 
```
 claude plugin marketplace  add git@github.com:amdchetanmurthy/ntsg-claude-plugins.git
 claude plugin install amd-ai-infra@ntsg-claude-plugins
 ```

## Development

### Plugin Structure

Each plugin follows the standard Claude Code plugin structure:

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json      # Plugin metadata (required)
├── .mcp.json            # MCP server configuration (optional)
├── commands/            # Slash commands (optional)
├── agents/              # Agent definitions (optional)
├── skills/              # Skill definitions (optional)
├── hooks/               # Event hooks (optional)
└── README.md            # Documentation
```

### Adding a New Plugin

1. Create a new directory under `plugins/`
2. Follow the standard plugin structure
3. Add plugin metadata to `.claude-plugin/plugin.json`
4. Document the plugin in its own README.md
5. Update this README to list the new plugin

## Resources

- [Claude Code Plugin Documentation](https://code.claude.com/docs/en/plugins)
- [Plugin Development Guide](https://code.claude.com/docs/en/plugins/development)
- [MCP Protocol](https://modelcontextprotocol.io/)

## License

See individual plugin directories for license information.
