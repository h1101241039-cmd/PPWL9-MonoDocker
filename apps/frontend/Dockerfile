FROM oven/bun:1 AS builder
WORKDIR /app
COPY package.json bun.lock ./
COPY apps/frontend/package.json ./apps/frontend/
COPY packages/shared/package.json ./packages/shared/
RUN bun install
COPY apps/frontend ./apps/frontend
COPY packages/shared ./packages/shared
WORKDIR /app/apps/frontend
RUN bun run build

FROM nginx:alpine
COPY --from=builder /app/apps/frontend/dist /usr/share/nginx/html
COPY apps/frontend/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]