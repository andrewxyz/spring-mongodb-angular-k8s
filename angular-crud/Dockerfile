# ---- Stage 1: build the Angular app ----
# Angular 17 supports Node 18/20
FROM node:20-alpine AS build
WORKDIR /src

# Install dependencies first so this layer is cached when only code changes
COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npx ng build --configuration production

# ---- Stage 2: serve the static files with nginx ----
FROM nginx:1.28-alpine
# nginx.conf is mounted from a ConfigMap in Kubernetes; this default config
# also works on its own (docker run) thanks to the try_files fallback below.
RUN printf 'server {\n  listen 80;\n  root /usr/share/nginx/html;\n  index index.html;\n  location / { try_files $uri $uri/ /index.html; }\n}\n' \
      > /etc/nginx/conf.d/default.conf
COPY --from=build /src/dist/angular-17-crud/browser /usr/share/nginx/html

EXPOSE 80
