# ---- Frontend (React + Vite + TypeScript) ----
# Confirmed: "build":"tsc -p tsconfig.prod.json && vite build" -> outputs to /dist.
# VITE_API_PATH is read at BUILD time and baked into the bundle, so it's a build-arg.
# NOTE: frontend package.json has a "prepare" script ("cd .. && npm install") meant for
# the monorepo root; in an isolated Docker context that parent doesn't exist, so we install
# with --ignore-scripts to skip it. No committed lockfile -> npm install, not npm ci.

# ---- Stage 1: build ----
FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm install --ignore-scripts

# The browser uses this URL to reach the backend (which runs on 8080):
#  - local compose:    http://localhost:8080
#  - on EKS (NodePort): http://<node-public-ip>:31100
ARG VITE_API_PATH
ENV VITE_API_PATH=${VITE_API_PATH}

COPY . .
RUN npm run build

# ---- Stage 2: serve ----
FROM nginx:1.27-alpine AS runtime
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
