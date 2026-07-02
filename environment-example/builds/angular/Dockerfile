FROM node:22-bookworm

# Evita preguntas durante la instalación
ENV DEBIAN_FRONTEND=noninteractive

# Herramientas básicas
RUN apt-get update && apt-get install -y \
    git \
    curl \
    bash \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Instalar Angular CLI globalmente
RUN npm install -g @angular/cli

# Directorio de trabajo
WORKDIR /workspace

# Puerto habitual del servidor de desarrollo
EXPOSE 4200

CMD ["bash"]