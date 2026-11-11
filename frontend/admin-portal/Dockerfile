# syntax=docker/dockerfile:1
FROM node:26.12-bookworm-slim AS build
WORKDIR /workspace
COPY package.json package-lock.json ./
RUN npm ci --no-audit --no-fund
COPY . .
RUN npx ng build --configuration production

# Unprivileged Nginx (listens on 8080, runs as uid 101).
FROM nginxinc/nginx-unprivileged:1.31-alpine
COPY docker/nginx.conf.template /etc/nginx/templates/default.conf.template
COPY docker/40-runtime-config.sh /docker-entrypoint.d/40-runtime-config.sh
COPY --from=build /workspace/dist/admin-portal/browser /usr/share/nginx/html
# The runtime config is generated at container start, so the web root's config.json must be writable.
USER root
RUN chown 101:101 /usr/share/nginx/html && rm -f /usr/share/nginx/html/config.json
USER 101
ENV API_BASE_URL=http://localhost:8080/api/v1 \
    OIDC_ISSUER=http://localhost:8180/realms/occpl \
    OIDC_CLIENT_ID=occpl-admin-portal \
    CSP_CONNECT_SRC="http://localhost:8080 http://localhost:8180"
EXPOSE 8080
HEALTHCHECK --interval=15s --timeout=3s CMD wget -qO- http://127.0.0.1:8080/healthz || exit 1
