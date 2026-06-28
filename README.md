# Tester

A small Unreal Engine **5.8** sandbox project created to try out Unreal's **experimental Model Context Protocol (MCP) feature**.

## Purpose

The whole point of this project is experimentation: to see how an AI assistant (via MCP)
can drive the Unreal Editor — spawning and arranging actors, creating materials, and
manipulating a level — rather than to build a finished game or application.

Unreal 5.8 ships an experimental **ModelContextProtocol** plugin that exposes editor
functionality over MCP. With it enabled, an MCP client (such as Claude Code) can connect
to a running editor instance and issue tool calls to build and modify content in the level.

## What's in here

- **`MCPTestMap`** — the test level used for the experiments.
- **`Content/StackedCubes/`** — assets generated/used during the experiments, including:
  - `BP_SphereTornado` — a Blueprint actor.
  - `M_CubeColor` — a base material, with a full set of colored material instances
    (`MI_Red`, `MI_Blue`, `MI_Green`, etc.) and their metallic variants (`MI_*Metal`).

This is a **Blueprint-only** project — there is no C++ source.

## MCP setup

The MCP connection is configured in [`.mcp.json`](.mcp.json):

```json
{
  "mcpServers": {
    "unreal-mcp": {
      "type": "http",
      "url": "http://127.0.0.1:8000/mcp"
    }
  }
}
```

The client connects over HTTP to a local MCP server (default `127.0.0.1:8000`) that is
exposed by the editor's `ModelContextProtocol` plugin while the project is open.

### Requirements

- Unreal Engine **5.8** (this project's `EngineAssociation`).
- The following plugins enabled (already set in `Tester.uproject`), notably:
  - **ModelContextProtocol** — exposes the MCP server.
  - Plus the editor tooling plugins used during the experiments
    (ModelingToolsEditorMode, MeshTerrainMode, Composite, AllToolsets, and others).

### Running

1. Open `Tester.uproject` in Unreal Engine 5.8.
2. Confirm the **ModelContextProtocol** plugin is enabled and the MCP server is listening
   on `127.0.0.1:8000`.
3. Connect your MCP client (e.g. Claude Code) using the config in `.mcp.json`.
4. Issue tool calls to inspect and modify `MCPTestMap`.

## Status

Experimental / throwaway sandbox. Expect rough edges — this exists to explore the MCP
feature, not to ship anything.
