---
name: discovery-test
description: >
  Probe skill used to verify that the Graffiticode MCP server picks up new
  skills from the graffiticode-skills repo dynamically, with no rebuild or
  redeploy. Not a real authoring skill — safe to remove once discovery has
  been confirmed.
---

# discovery-test

This file exists only to validate runtime skill discovery. If you can read it
through the Graffiticode MCP as the resource `graffiticode://skills/discovery-test`,
then the server is fetching skills live from the public `graffiticode-skills`
repository on GitHub — adding this directory required no change to the server.

It carries no agent guidance and can be deleted at any time.
