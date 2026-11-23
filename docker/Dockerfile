FROM node:26-slim@sha256:e6bedef217677b276c591795e169840edd9096dbf13f29df8ef8d1856b8820bc AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY src/ src/
COPY static/ static/
COPY svelte.config.js vite.config.ts tsconfig.json ./
COPY drizzle/ drizzle/
COPY scripts/ scripts/
# The SvelteKit build validates DATABASE_URL (src/lib/server/config.ts) during
# the postbuild route-analysis step, but never connects to a database. Pass it
# as an optional BuildKit secret so it is not persisted in the image. When no
# secret is supplied (e.g. CI) fall back to a throwaway dummy value.
RUN --mount=type=secret,id=db_url \
    DATABASE_URL="$(cat /run/secrets/db_url 2>/dev/null || echo 'postgres://build:build@localhost:5432/build')" \
    npm run build
# node_modules is copied into the runtime image whole, because the seed and
# migrate scripts need drizzle-orm and postgres. drizzle-kit and svelte-i18n
# each bring their own old esbuild, a Go binary the image scan fails on and
# nothing uses at runtime (svelte-i18n only needs it for its CLI). Drop those
# copies; the current esbuild that tsx uses stays.
RUN rm -rf node_modules/drizzle-kit/node_modules/esbuild \
           node_modules/drizzle-kit/node_modules/@esbuild \
           node_modules/@esbuild-kit/core-utils/node_modules/esbuild \
           node_modules/@esbuild-kit/core-utils/node_modules/@esbuild \
           node_modules/svelte-i18n/node_modules/esbuild \
           node_modules/svelte-i18n/node_modules/@esbuild

FROM node:26-slim@sha256:e6bedef217677b276c591795e169840edd9096dbf13f29df8ef8d1856b8820bc
# The base image lags Debian's security fixes; the image scan fails on a
# fixable critical, so take the fixes that exist at build time.
RUN apt-get update && apt-get upgrade -y && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
# Nothing in this image runs npm: the entrypoint starts `node` directly. The
# npm the base image ships carries its own copies of undici and
# brace-expansion, which the image scan fails on and no update here can reach.
# Drop it, with the npx and corepack launchers that point into it.
RUN rm -rf /usr/local/lib/node_modules/npm /usr/local/lib/node_modules/corepack \
           /usr/local/bin/npm /usr/local/bin/npx /usr/local/bin/corepack
WORKDIR /app
COPY --from=build /app/build ./build
COPY --from=build /app/package.json ./
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/drizzle/migrations ./drizzle/migrations
COPY --from=build /app/src/lib/server/db/schema.ts ./src/lib/server/db/schema.ts
COPY --from=build /app/scripts ./scripts
COPY docker/entrypoint.sh ./entrypoint.sh
ENV NODE_ENV=production
ENV PORT=3000
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
ENTRYPOINT ["./entrypoint.sh"]
