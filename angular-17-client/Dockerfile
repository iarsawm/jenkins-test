### Multi-stage Dockerfile for Angular frontend
#  - Stage 1: build with Node
#  - Stage 2: serve static files with nginx

FROM node:18-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --silent
COPY . .
# build production bundle (adjust flags if your project uses a different configuration)
RUN npm run build -- --configuration=production

FROM nginx:stable-alpine
COPY --from=build /app/dist /usr/share/nginx/html
# Optional: replace default nginx config for SPA routing
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
