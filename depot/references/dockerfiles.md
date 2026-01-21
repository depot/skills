# Optimal Dockerfile Patterns

Language-specific Dockerfile patterns optimized for Depot's BuildKit cache mounts.

## Table of Contents
- [Node.js (npm)](#nodejs-npm)
- [Node.js (pnpm)](#nodejs-pnpm)
- [Python (pip)](#python-pip)
- [Go](#go)
- [Rust](#rust)
- [Ruby (Bundler)](#ruby-bundler)
- [PHP (Composer)](#php-composer)
- [Java (Maven/Gradle)](#java)
- [APT Package Installation](#apt-packages)

---

## Node.js (npm)

```dockerfile
FROM node:22-slim AS build
WORKDIR /app

# Install dependencies with cache mount
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Build application
COPY . .
RUN npm run build

# Production stage
FROM node:22-slim AS runtime
ENV NODE_ENV=production
WORKDIR /app

# Create non-root user
RUN groupadd -g 1001 appgroup && \
    useradd -u 1001 -g appgroup -m -d /app -s /bin/false appuser

# Copy built application
COPY --from=build --chown=appuser:appgroup /app/dist ./dist
COPY --from=build --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=build --chown=appuser:appgroup /app/package.json ./

USER appuser
CMD ["node", "dist/index.js"]
```

---

## Node.js (pnpm)

```dockerfile
FROM node:22-slim AS build
RUN corepack enable
WORKDIR /app

COPY package.json pnpm-lock.yaml ./
RUN --mount=type=cache,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile

COPY . .
RUN pnpm run build

# Production dependencies only
FROM node:22-slim AS deps
RUN corepack enable
WORKDIR /app

COPY package.json pnpm-lock.yaml ./
RUN --mount=type=cache,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile --prod

# Runtime
FROM node:22-slim AS runtime
ENV NODE_ENV=production
WORKDIR /app

RUN groupadd -g 1001 appgroup && \
    useradd -u 1001 -g appgroup -m -d /app -s /bin/false appuser

COPY --from=build --chown=appuser:appgroup /app/dist ./dist
COPY --from=deps --chown=appuser:appgroup /app/node_modules ./node_modules

USER appuser
CMD ["node", "dist/index.js"]
```

---

## Python (pip)

```dockerfile
FROM python:3.13-slim AS build
RUN pip install --upgrade pip setuptools wheel
WORKDIR /app

# Create virtual environment
RUN python -m venv .venv
ENV PATH="/app/.venv/bin:$PATH"

# Install dependencies with cache mount
COPY requirements.txt ./
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

COPY . .

# Runtime
FROM python:3.13-slim AS runtime
ENV PATH="/app/.venv/bin:$PATH"
WORKDIR /app

RUN groupadd -g 1001 appgroup && \
    useradd -u 1001 -g appgroup -m -d /app -s /bin/false appuser

COPY --from=build --chown=appuser:appgroup /app .

USER appuser
ENTRYPOINT ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Python with Poetry

```dockerfile
FROM python:3.13-slim AS build
RUN pip install poetry
WORKDIR /app

COPY pyproject.toml poetry.lock ./
RUN --mount=type=cache,target=/root/.cache/pypoetry \
    poetry config virtualenvs.in-project true && \
    poetry install --no-dev --no-interaction

COPY . .

FROM python:3.13-slim AS runtime
ENV PATH="/app/.venv/bin:$PATH"
WORKDIR /app
COPY --from=build /app .
CMD ["python", "-m", "app"]
```

### Python with uv (fastest)

```dockerfile
FROM python:3.13-slim AS build
RUN pip install uv
WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

COPY . .

FROM python:3.13-slim AS runtime
ENV PATH="/app/.venv/bin:$PATH"
WORKDIR /app
COPY --from=build /app .
CMD ["python", "-m", "app"]
```

---

## Go

```dockerfile
FROM golang:1.23 AS build
WORKDIR /app

# Download dependencies with cache mount
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

# Build with build cache
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -o /app/server ./cmd/server

# Minimal runtime
FROM gcr.io/distroless/static-debian12 AS runtime
COPY --from=build /app/server /server
ENTRYPOINT ["/server"]
```

---

## Rust

### Basic with cache mounts

```dockerfile
FROM rust:1.83 AS build
WORKDIR /app

COPY . .
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/usr/local/cargo/git \
    --mount=type=cache,target=/app/target \
    cargo build --release && \
    cp target/release/myapp /myapp

FROM debian:bookworm-slim AS runtime
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=build /myapp /usr/local/bin/myapp
ENTRYPOINT ["myapp"]
```

### Optimized with cargo-chef and sccache (75% faster)

```dockerfile
FROM rust:1.83 AS base
RUN cargo install sccache --version ^0.8
RUN cargo install cargo-chef --version ^0.1
ENV RUSTC_WRAPPER=sccache SCCACHE_DIR=/sccache

FROM base AS planner
WORKDIR /app
COPY . .
RUN --mount=type=cache,target=$SCCACHE_DIR,sharing=locked \
    cargo chef prepare --recipe-path recipe.json

FROM base AS builder
WORKDIR /app
COPY --from=planner /app/recipe.json recipe.json

# Build dependencies (cached separately from app code)
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=$SCCACHE_DIR,sharing=locked \
    cargo chef cook --release --recipe-path recipe.json

# Build application
COPY . .
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=$SCCACHE_DIR,sharing=locked \
    cargo build --release

FROM debian:bookworm-slim AS runtime
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp
ENTRYPOINT ["myapp"]
```

---

## Ruby (Bundler)

```dockerfile
FROM ruby:3.3-slim AS build
WORKDIR /app

RUN apt-get update && apt-get install -y build-essential git

COPY Gemfile Gemfile.lock ./
RUN --mount=type=cache,target=/usr/local/bundle/cache \
    bundle install --jobs 4 --retry 3

COPY . .

FROM ruby:3.3-slim AS runtime
WORKDIR /app
COPY --from=build /usr/local/bundle /usr/local/bundle
COPY --from=build /app .
CMD ["bundle", "exec", "rails", "server"]
```

---

## PHP (Composer)

```dockerfile
FROM php:8.3-fpm AS build
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
WORKDIR /app

COPY composer.json composer.lock ./
RUN --mount=type=cache,target=/root/.composer/cache \
    composer install --no-dev --no-scripts --prefer-dist

COPY . .
RUN composer dump-autoload --optimize

FROM php:8.3-fpm AS runtime
WORKDIR /app
COPY --from=build /app .
```

---

## Java

### Maven

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app

COPY pom.xml ./
RUN --mount=type=cache,target=/root/.m2/repository \
    mvn dependency:go-offline

COPY src ./src
RUN --mount=type=cache,target=/root/.m2/repository \
    mvn package -DskipTests

FROM eclipse-temurin:21-jre AS runtime
COPY --from=build /app/target/*.jar /app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### Gradle

```dockerfile
FROM gradle:8-jdk21 AS build
WORKDIR /app

COPY build.gradle settings.gradle ./
RUN --mount=type=cache,target=/home/gradle/.gradle/caches \
    gradle dependencies --no-daemon

COPY src ./src
RUN --mount=type=cache,target=/home/gradle/.gradle/caches \
    gradle build -x test --no-daemon

FROM eclipse-temurin:21-jre AS runtime
COPY --from=build /app/build/libs/*.jar /app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

---

## APT Packages

Always use `sharing=locked` for apt since it doesn't tolerate concurrent access:

```dockerfile
RUN --mount=type=cache,sharing=locked,target=/var/cache/apt \
    --mount=type=cache,sharing=locked,target=/var/lib/apt \
    apt-get update && \
    apt-get install -y --no-install-recommends \
        build-essential \
        curl \
        git && \
    rm -rf /var/lib/apt/lists/*
```

---

## Key Principles

1. **Cache mounts persist across builds** - No need for `--cache-from`/`--cache-to` with Depot
2. **Use multi-stage builds** - Separate build and runtime for smaller images
3. **Order by change frequency** - Dependencies before source code
4. **Use slim/alpine bases** - Smaller images, faster pulls
5. **Run as non-root** - Security best practice
6. **Avoid COPY --link** - Has caching bugs, traditional COPY is often faster
