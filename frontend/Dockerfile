# Multi-stage build for frontend
FROM node:18-alpine AS build

WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm i --silent

# Copy source code and build
COPY . .
RUN npm run build

# Production stage with NGINX
FROM nginx:alpine AS production

# Copy built app to nginx directory
COPY --from=build /app/build /usr/share/nginx/html

# Copy custom nginx config
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Create non-root user for nginx
RUN addgroup -g 1001 -S nginx-app && \
    adduser -S nginx-app -u 1001

# Change ownership of nginx directories
RUN chown -R nginx-app:nginx-app /usr/share/nginx/html && \
    chown -R nginx-app:nginx-app /var/cache/nginx && \
    chown -R nginx-app:nginx-app /var/log/nginx && \
    chown -R nginx-app:nginx-app /etc/nginx/conf.d && \
    touch /var/run/nginx.pid && \
    chown -R nginx-app:nginx-app /var/run/nginx.pid

USER nginx-app

EXPOSE 4040

CMD ["nginx", "-g", "daemon off;"]