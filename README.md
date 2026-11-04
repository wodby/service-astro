# Node.js (Astro) service for Kubernetes on Wodby

Build and run Astro applications on Kubernetes with Wodby.

This repository defines the Wodby service manifests and operational
configuration for Node.js (Astro).

- [Browse Wodby services](https://wodby.com/services)
- [Wodby service documentation](https://wodby.com/docs/2.0/services/)
- [Service manifest reference](https://wodby.com/docs/2.0/services/template/)

## Start with a boilerplate

Use one of the boilerplates exposed by this service to start with compatible
build configuration and Wodby CI:

- [Astro Boilerplate](https://github.com/wodby/astro-boilerplate)

## Wodby stacks using this service

- [Astro application stack](https://github.com/wodby/stack-astro)

## Service overview

| Property | Manifest configuration |
| --- | --- |
| Service name | `astro` |
| Type | Application service |
| Inherits from | [`node`](https://github.com/wodby/service-node) with version constraint `^2.0.0` |
| Application build | Git source connection enabled; Dockerfile: `Dockerfile`; boilerplates: Astro Boilerplate |

## Use this service

Use this service through [Astro application stack](https://github.com/wodby/stack-astro), or reference
`astro` from a custom Wodby stack.

A service is a reusable component and does not deploy by itself. The stack
defines its links, settings, versions, resources, and relationship to the rest
of the application.

The image build runs `npm run build` and the container starts with
`npm run start`. Applications use the standalone server of Astro's Node.js
adapter: `node({ mode: 'standalone' })` and `node ./dist/server/entry.mjs`.
The service sets `HOST=0.0.0.0` and `PORT=3000` for that server.

## Maintain a custom version

1. Fork this repository.
2. Edit the service manifest and referenced files.
3. Import the repository as a [Git-backed service](https://wodby.com/docs/2.0/services/create/#create-a-git-backed-service).
4. Reference the service from a stack manifest.

Keep service, workload, container, endpoint, link, volume, config, and
derivative names stable unless dependent stacks and app-level overrides are
updated at the same time.

Validate the manifests with:

```bash
wodby service validate-manifest service.yml --org <org-id> --include GUIDANCE.md
```

See the [service manifest reference](https://wodby.com/docs/2.0/services/template/) and the [managed services index](https://github.com/wodby/services).

## Workspace startup

Workspaces run `astro dev --host "$HOST" --port "$PORT"` from the project's
`node_modules`, with the project's own Astro configuration. This command does
not run custom `dev`, `predev` or `postdev` scripts. Override
`WORKSPACE_NODE_COMMAND` when those scripts are required, and configure that
command's watcher for shared storage. Add the environment's hosts to
`server.allowedHosts` in the Astro configuration. Vite's watcher polls because
the workspace sets `CHOKIDAR_USEPOLLING`. Standard build/start behavior is
unchanged. Requires a Node image with workspace contract 1.
