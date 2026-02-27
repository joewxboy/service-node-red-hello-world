# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an Open Horizon service container that deploys [Node-RED](https://nodered.org) to edge nodes. It packages a simple "Hello World" Node-RED flow into a multi-arch Docker container (amd64 and arm64) and publishes it as an Open Horizon service.

Node-RED runs on port 1880. The flow in `flow.json` injects a "Hello World" message every 10 seconds to a debug node.

## Key Commands

```sh
make build          # Build multi-arch Docker images (amd64 and arm64)
make run            # Run container locally on port 1880
make stop           # Stop the locally-running container
make test           # Verify Node-RED is running (curl /diagnostics)
make browse         # Open http://127.0.0.1:1880/ in a browser
make dev            # Run container and attach a shell inside it
make clean          # Remove container images
make log            # View Open Horizon event and service logs
make deploy-check   # Check agent/service policy compatibility
```

## Open Horizon Deployment

```sh
make publish        # Publish service + policies, then register the agent
make agent-stop     # Unregister agent and stop all agreements
make distclean      # Full cleanup: agent-stop + remove policies + clean images
```

Required environment variables for Open Horizon operations:
- `HZN_ORG_ID` – your org (default: `examples`)
- `HZN_EXCHANGE_USER_AUTH` – credentials for the exchange
- `DOCKER_HUB_ID` – Docker Hub namespace (default: `docker.io/walicki`)

## Architecture

### Container Build (`Dockerfile`)
Two-stage build using Red Hat UBI9:
1. **Build stage** (`ubi9:9.1.0-1646`): Installs Node.js/npm, runs `npm install` for Node-RED dependencies from `package.json`.
2. **Runtime stage** (`ubi9/ubi-minimal:9.1.0-1656`): Minimal image, runs as non-root user 1000. Copies `/opt/app-root/data` from build stage.

Artifacts copied into the image:
- `package.json` → Node-RED dependencies
- `settings.js` → Node-RED runtime configuration
- `flow.json` → Node-RED flow (becomes `flows.json` inside container)
- `flow_cred.json` → Flow credentials (becomes `flows_cred.json` inside container)

### Open Horizon Integration (`horizon/`)
- `service.definition.json` – Service definition with image reference and port mapping; uses `$HZN_ORG_ID`, `$SERVICE_NAME`, `$SERVICE_VERSION`, `$ARCH`, and `$CONTAINER_IMAGE_BASE` variable substitution
- `service.policy.json` – Service-side policy constraints
- `deployment.policy.json` – Deployment policy; constraint `nodered == helloworld` matches node policy
- `node.policy.json` – Edge node policy; property `nodered=helloworld` triggers deployment
- `userinput.json` – User input variables for the service
- `servicesecret` – Secret binding file

Policy matching: The node policy sets `nodered=helloworld`, and the deployment policy constrains `nodered == helloworld`, causing Open Horizon to automatically deploy the service to matching nodes.

### Modifying the Flow
Edit `flow.json` directly or use a local Node-RED editor with [Node-RED Projects](https://nodered.org/docs/user-guide/projects/) enabled. Add npm package dependencies to `package.json` before building.
