# syntax=docker/dockerfile:1

#############################################################
# Stage 1 - Build the React application
#############################################################
FROM node:24-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

#############################################################
# Stage 2 - Runtime: serve the built SPA with Nginx
#############################################################
FROM nginx:1.31.3-alpine AS runtime

# Custom Nginx config (SPA routing, API proxy, security headers)
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy compiled assets from the builder stage only
COPY --from=builder /app/dist /usr/share/nginx/html

# Create dirs writable by nginx user (UID 101)
# Kubernetes securityContext enforces runAsUser: 101 (nginx).
RUN mkdir -p /run /var/cache/nginx/client_temp && chown -R nginx:nginx /run /var/cache/nginx

# Drop worker privileges after startup (nginx master stays root, workers run as nginx)
USER nginx

EXPOSE 80

# Non-shell healthcheck so the Kubernetes liveness/readiness works from the start
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -q -O - http://127.0.0.1/health || exit 1

STOPSIGNAL SIGQUIT

CMD ["nginx", "-g", "daemon off;"]