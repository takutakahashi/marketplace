# takutakahashi's Claude Code Plugin Marketplace

Custom Claude Code plugins and skills for agentapi and related development tools.

## Installation

Add this marketplace to Claude Code:

```bash
/plugin marketplace add takutakahashi/marketplace
```

## Available Plugins

### agentapi-proxy-api

Comprehensive plugin for interacting with the agentapi-proxy API using API Key authentication.

**Install:**
```bash
/plugin install agentapi-proxy-api@takutakahashi-plugins
```

**Features:**
- Complete API endpoint reference with examples
- Multiple authentication methods (X-API-Key, Bearer token)
- Detailed permission and RBAC documentation
- Helper script for making API requests
- Configuration examples

**Included Resources:**
- **SKILL.md**: Core workflow and quick start guide
- **references/API_REFERENCE.md**: Complete endpoint documentation
- **references/AUTHENTICATION.md**: Authentication methods and setup
- **references/PERMISSIONS.md**: Role-based access control details
- **scripts/agentapi_request.sh**: Helper script for API requests
- **assets/**: Configuration examples

## Usage

After installation, Claude Code will have knowledge of:
- agentapi-proxy API endpoints and usage
- API Key authentication methods
- Session management workflows
- Permission and role configurations

Example interactions:
- "How do I create a new agentapi session?"
- "Show me how to authenticate with an API key"
- "What permissions does the user role have?"

## Contributing

To add new plugins to this marketplace:

1. Create a plugin directory under `plugins/`
2. Add `.claude-plugin/plugin.json` with plugin metadata
3. Add your skill(s) to `skills/` subdirectory
4. Update `.claude-plugin/marketplace.json` to include the new plugin
5. Submit a pull request

## License

MIT