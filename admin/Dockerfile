# ClawForge Admin Console — multi-stage build
FROM node:22-alpine AS base
WORKDIR /app

# Install dependencies
FROM base AS deps
COPY package.json package-lock.json* ./
RUN npm install

# Build Next.js (standalone output)
FROM deps AS build
ARG NEXT_PUBLIC_API_URL=http://localhost:4100
ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL
COPY . .
RUN npm run build

# Production image
FROM base AS runner
ENV NODE_ENV=production

# Next.js standalone output includes everything needed
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public

EXPOSE 4200
ENV PORT=4200
CMD ["node", "server.js"]
