# Base image
FROM node:20-slim AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
LABEL fly_launch_runtime="Node.js"
RUN corepack enable
WORKDIR /app

# Install production dependencies
FROM base AS prod-deps
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --prod --frozen-lockfile

# Build stage
FROM base AS build
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
COPY . .

# Install certificates
RUN apt-get update -qq && apt-get install -y ca-certificates && update-ca-certificates

# Build the application
# Remove the unsupported secret mount and adjust the build command
ARG SENTRY_AUTH_TOKEN
RUN if [ -z "$SENTRY_AUTH_TOKEN" ]; then \
        pnpm run build:nosentry; \
    else \
        pnpm run build; \
    fi

# Build Go shared library
FROM golang:1.19 AS go-build
WORKDIR /app/src/lib/go-html-to-md
COPY src/lib/go-html-to-md .
RUN go mod tidy && \
    go build -o html-to-markdown.so -buildmode=c-shared html-to-markdown.go && \
    chmod +x html-to-markdown.so

# Final image
FROM base
COPY --from=prod-deps /app/node_modules /app/node_modules
COPY --from=build /app /app
COPY --from=go-build /app/src/lib/go-html-to-md/html-to-markdown.so /app/dist/src/lib/go-html-to-md/html-to-markdown.so

# Install Chromium for Puppeteer
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y chromium chromium-sandbox && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

# Set environment variables
EXPOSE 8080
ENV PUPPETEER_EXECUTABLE_PATH="/usr/bin/chromium"

# Ensure the entrypoint script has correct line endings
RUN sed -i 's/\r$//' /app/docker-entrypoint.sh

ENTRYPOINT ["/app/docker-entrypoint.sh"]
