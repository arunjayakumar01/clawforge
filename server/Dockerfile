# ClawForge Server — multi-stage build
FROM node:22-alpine AS base
WORKDIR /app

# Install dependencies
FROM base AS deps
COPY package.json package-lock.json* ./
RUN npm install

# Build TypeScript
FROM deps AS build
COPY . .
RUN npm run build

# Production image
FROM base AS runner
ENV NODE_ENV=production

COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json ./package.json
# Drizzle needs the config + migrations at runtime for db:migrate
COPY --from=build /app/drizzle.config.ts ./drizzle.config.ts
COPY --from=build /app/src/db/migrations ./src/db/migrations
COPY --from=build /app/src/db/schema.ts ./src/db/schema.ts
# Seed script (tsx runs TS directly)
COPY --from=build /app/src/db/seed.ts ./src/db/seed.ts

EXPOSE 4100
CMD ["node", "dist/index.js"]
