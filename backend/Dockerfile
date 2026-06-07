# ---- Backend (Node + TypeScript + Express, ESM) ----
# Confirmed from package.json: "type":"module", "build":"tsc", "start":"npm run build && node dist/server.js".
# So we compile in a builder stage and run dist/server.js directly in the runtime stage
# (running `npm start` would re-run tsc at boot, but the compiler isn't in the runtime image).
# No committed package-lock.json in the repo, so we use `npm install`, not `npm ci`.

# ---- Stage 1: build ----
FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build            # tsc -> dist/

# ---- Stage 2: runtime ----
FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production

# package.json carries "type":"module", so Node treats dist/*.js as ESM. Keep it.
COPY package*.json ./
RUN npm install --omit=dev && npm cache clean --force

COPY --from=builder /app/dist ./dist

# App listens on 8080 (backend/.env.sample: PORT=8080).
EXPOSE 8080

CMD ["node", "dist/server.js"]