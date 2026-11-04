# Astro on Wodby

What this service adds to the Node.js service it is based on.

## Build and start

The service's Dockerfile copies the build context to `/usr/src/app`, runs `npm run build` and starts the container with `npm run start`. `package.json` must define both scripts.

- The supported production output is the standalone server of the `@astrojs/node` adapter: `adapter: node({ mode: 'standalone' })` in `astro.config.mjs`, `astro build`, and `node ./dist/server/entry.mjs` as the start script. The standalone server renders on-demand pages and serves prerendered pages, `public/` files and built assets from `dist/client`. Static output without an adapter has no server to start, and `mode: 'middleware'` needs a server of its own.
- Astro 7 needs Node.js 22.23.4 or later. All Node.js versions of the service meet it.
- Pages are prerendered at build time unless they set `export const prerender = false` or the configuration sets `output: 'server'`. Endpoints, Actions submitted by forms, and pages that read request data, cookies or runtime variables must be rendered on demand.
- The Dockerfile does not install dependencies. The pipeline does it before the image is built (`wodby ci run -- npm ci`), and `node_modules` is copied with the code. Build tools such as `@astrojs/check` and TypeScript may stay in `devDependencies`.
- `npm run build` runs while the image is built. Runtime variables of the service, including the link variables and secrets, are not available to it. Prerendered pages are built without them.
- The image build leaves out `.git`, `.gitignore`, `.dockerignore`, `.wodby-ci-cache`, `dist` and `.astro`.
- The standalone server has no handler for `SIGTERM`: the process ends at once when the container is stopped, and requests in progress are cut off.

## Variables the service sets

| Variable | Meaning |
| --- | --- |
| `HOST` | `0.0.0.0`. The standalone server listens on it; without it, it listens on `localhost` only and the gateway cannot reach it. |
| `PORT` | `3000`, the port of the service's endpoint. The standalone server reads it at startup; it takes precedence over `server.port` of the configuration. |
| `NODE_ENV` | `production` in every environment type, including `dev`. The Node.js service's `development` value for `dev` is replaced, because the container runs a production build. |
| `WORKSPACE_NODE_COMMAND` | `exec node node_modules/astro/bin/astro.mjs dev --host "$HOST" --port "$PORT"`, the start command of a development workspace. |

## Runtime variables

- Read runtime variables on the server when a request is handled: with `process.env`, or with `astro:env` fields declared with `context: 'server'` and `access: 'secret'` and imported from `astro:env/server`. Secrets are validated when the module is first imported at runtime, not during the build, unless `env.validateSecrets` is `true`.
- `import.meta.env` and `astro:env` fields with `access: 'public'` are replaced with their build-time values. They cannot hold runtime variables or secrets. Only `PUBLIC_` variables and `context: 'client'` fields reach the browser.
- `astro.config.mjs` is evaluated during the build and in the development server. It reads `process.env`, not `astro:env`.

## Linked services

The service adds no link variables. The database, mail and Redis or Valkey variables of the Node.js service apply unchanged. On-demand pages, endpoints, Actions and middleware read them at request time. They do not reach the browser.

The adapter's default session storage is the container's file system (`node_modules/.astro/sessions`). It is lost on every deployment and not shared between replicas. Configure a Redis session driver with the Redis or Valkey link before using `Astro.session`.

## Domains, forwarded headers and form submissions

- The gateway terminates TLS, keeps the original `Host` header and sets `X-Forwarded-Proto`. Astro uses `Host`, `X-Forwarded-Proto`, `X-Forwarded-Host` and `X-Forwarded-For` only for domains matched by `security.allowedDomains`. Without a match, `Astro.url` and `Astro.request.url` are `http://localhost:3000/...`.
- The domains are in `WODBY_HOSTS` at runtime but not during the build, where `security.allowedDomains` is compiled in. The boilerplate sets `allowedDomains: [{}]` (every domain) and rejects, in `src/middleware.ts`, any request whose `X-Forwarded-Host` is not one of `WODBY_HOSTS`. Keep both together: `[{}]` alone lets a client choose the host of `Astro.url` with `X-Forwarded-Host`.
- With that configuration, `Astro.clientAddress` is the first `X-Forwarded-For` value, which the client can set. Do not use it for access control.
- `security.checkOrigin` is on by default: form-encoded `POST`, `PUT`, `PATCH` and `DELETE` requests to on-demand routes and Actions are rejected with 403 when `Sec-Fetch-Site` is anything but `same-origin` or `none`, or, without it, when `Origin` differs from the origin of `Astro.url`. That comparison needs the forwarded `https` protocol and host above.

## In a development workspace

- The application is started with `astro dev` from `node_modules`, not through the `dev` script of `package.json`. Options in that script do not apply. `astro` must be a dependency of the project.
- The development server listens on `0.0.0.0:3000`. The command line sets both; `server.host` and `server.port` of the configuration do not apply.
- The development server rejects requests for host names it does not know. Add the environment's hosts to `server.allowedHosts` in `astro.config.mjs`: `WODBY_HOSTS` holds them as a JSON array, and `WODBY_APP_SERVICE_NAME` is the internal name. The boilerplate's `astro.config.mjs` does this.
- File watching uses polling: Vite's watcher reads `CHOKIDAR_USEPOLLING` and `CHOKIDAR_INTERVAL`, which the workspace sets. Saved files are picked up within seconds; new files are picked up once the storage reports the directory change, which can take longer. No configuration change is needed.
- `NODE_ENV` is `development` here.
- `/dist/` and `/.astro/` are kept out of Git status, in addition to the Node.js service's paths. Astro's build cache is in `node_modules/.astro`.
- The image build's `npm run build` does not run in a workspace.
