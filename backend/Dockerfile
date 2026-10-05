# ---------- base: shared starting point ----------
FROM node:24-bookworm-slim AS base
WORKDIR /app

# ---------- deps: install ALL dependencies (cached until the lockfile changes) ----------
FROM base AS deps
COPY package.json package-lock.json ./
RUN npm ci

# ---------- dev: used by docker compose for local development ----------
FROM base AS dev
ENV NODE_ENV=development
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["npm", "run", "start:dev"]

# ---------- build: compile TypeScript, then drop devDependencies ----------
FROM base AS build
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build && npm prune --omit=dev

# ---------- prod: small final image, only what is needed to run ----------
FROM base AS prod
ENV NODE_ENV=production
COPY --from=build --chown=node:node /app/package.json ./
COPY --from=build --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
