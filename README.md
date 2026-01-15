# FT Optix cheat sheet

Collection of C# snippets ready for copy-paste

## Disclaimer

Rockwell Automation maintains these repositories as a convenience to you and other users. Although Rockwell Automation reserves the right at any time and for any reason to refuse access to edit or remove content from this Repository, you acknowledge and agree to accept sole responsibility and liability for any Repository content posted, transmitted, downloaded, or used by you. Rockwell Automation has no obligation to monitor or update Repository content

The examples provided are to be used as a reference for building your application and should not be used in production as-is. It is recommended to adapt the example for the purpose, of observing the highest safety standards.

> [!IMPORTANT]
> This guide does not replace the official FT Optix documentation, it is just a place some code snippets with a brief explanation.

> [!WARNING]
> These snippets *may* use some non-public APIs that may be subject to changes, please refer to official documentation to access the publicly available APIs which are guaranteed to be maintained.

> [!WARNING]
> Some of the snippets from this repository may irremediably break your project, use them at your own risk. Make sure to implement proper error handling and testing before deploying them to a production environment.

> [!WARNING]
> Usage of the version control features of the FactoryTalk Optix IDE is highly recommended to avoid any data loss. Make sure to commit your changes regularly to recover from any potential issue.

## Tips

### NetLogic cheat sheet MCP server

> [!WARNING]
> This service is provided by GitMCP ([Website](https://gitmcp.io) | [GitHub](https://github.com/idosal/git-mcp)) and is not affiliated with Rockwell Automation.

You can configure your GitHub Copilot (or any AI agent supporting MCP) to use this repository as a custom cheat sheet for FactoryTalk Optix NetLogic with GitMCP, just follow these steps:

1. Open your AI agent settings
2. Locate the MCP server configuration section
3. Add the following URL as a new MCP server: `https://gitmcp.io/FactoryTalk-Optix/NetLogic_CheatSheet` and set the type to `sse`
4. Save the settings and restart your AI agent if necessary

This can significantly improve the quality of the code suggestions related to FactoryTalk Optix NetLogic when using your AI agent (for example GitHub Copilot in both Visual Studio and Visual Studio Code).

> [!TIP]
> Browse to [this link](https://gitmcp.io/FactoryTalk-Optix/NetLogic_CheatSheet) to see additional configuration instructions for your AI agent or software.

## Sections

### Introduction

- [The InformationModel](./pages/information-model.md)
- [NetLogic overview](./pages/netlogic-overview.md)
- [Accessing project nodes](./pages/accessing-project-nodes.md)
- [General FT Optix good practices](./pages/good-practices.md)

### General NetLogic tips and tricks

- [Objects, types and instances](./pages/creating-objects.md)
- [Sessions](./pages/sessions.md)
- [Asynchronous tasks](./pages/async-tasks.md)

### Variables

- [Managing aliases](./pages/managing-aliases.md)
- [Resource URI](./pages/resource-uri.md)
- [Variables formatting](./pages/variables-formatting.md)
- [Variables properties](./pages/variables-properties.md)
- [RegEx](./pages/regex.md)
- [Variables Interaction](./pages/variables-interaction.md)
- [Dynamic Links](./pages/dynamic-links.md)
- [Audit signature](./pages/audit-signature.md)

### OPC/UA

- [OPC/UA](./pages/opcua.md)
- [Companion Specifications](./pages/companion-specs.md)

### UI

- [DialogBox](./pages/dialog-boxes.md)
- [Events handling](./pages/events.md)
- [Colors](./pages/colors.md)
- [Advanced SVG](./pages/advanced-svg.md)
- [DataGrid](./pages/datagrids.md)
- [Animations](./pages/ui-animations.md)
- [Trends](./pages/trends.md)

### Databases

- [Database interaction](./pages/database-interaction.md)
- [Queries](./pages/queries.md)

### Others

- [Log output](./pages/log-output.md)
- [Random generation](./pages/random-generation.md)
- [Running commands](./pages/execute-command.md)
- [Locales and Translations](./pages/translations.md)
- [Users and groups](./pages/users-groups.md)
- [Alarms](./pages/alarming.md)
- [System Node](./pages/system-node.md)
- [Template Library](./pages/template-library.md) (starting from FactoryTalk Optix 1.6.X)
- [Reports](./pages/reports.md)
- [Recipes](./pages/recipes.md)
- [Serial port](./pages/serial-port.md)

### Communication drivers

- [PLC Tags](./pages/plc-tags.md)
- [Runtime tags import](./pages/runtime-tags-import.md)
- [Designtime tags import](./pages/designtime-tags-import.md) (starting from FactoryTalk Optix 1.7.X)

### Advanced

> [!WARNING]
> The following topics are very advanced and potentially dangerous, make sure you know what you're doing before implementing them as they could lead to unexpected results or even break your project.

- [Register OPC/UA observers](./pages/register-observers.md)
- [Dispatch OPC/UA events](./pages/dispatch-events.md)
- [Manipulate Optix nodes](./pages/manipulate-nodes.md)

### Extras

> [!TIP]
> These snippets are not directly involved with FactoryTalk Optix C# APIs but can be useful in some scenarios

- [FactoryTalk Optix Studio CLI](./pages/fto-studio-cli.md) (starting from FactoryTalk Optix 1.6.X)
- [Debugging tips](./pages/debugging-tips.md)
