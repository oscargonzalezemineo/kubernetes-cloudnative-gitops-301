# ===========================
# Etapa 1: Build
# ===========================
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

RUN npm run build

# ===========================
# Etapa 2: Producción
# ===========================
FROM nginx:alpine

# Sustituye "mi-app" por el nombre de tu proyecto Angular
COPY --from=builder /app/dist/mi-app/browser /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]