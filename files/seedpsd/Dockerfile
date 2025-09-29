#############################################
FROM python:3.11-slim AS builder
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Enable bytecode compilation
# Copy from the cache instead of linking since it's a mounted volume
ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy

WORKDIR /app
RUN touch README.md

# Sync only project dependencies
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --locked --no-install-project --no-dev

# Bring in the project source
COPY . /app

# Sync project
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev

#############################################
FROM python:3.11-slim AS seedpsd

# Create unprivileged user
RUN addgroup --system app && \
    adduser --system --group app

# Copy from builder as root first (avoids BuildKit --chown edge cases),
# then fix ownership explicitly.
COPY --from=builder /app /app
RUN chown -R app:app /app

# Drop privileges
USER app

# Place executables in the environment at the front of the path
ARG CI_COMMIT_SHORT_SHA
ENV SENTRY_RELEASE=$CI_COMMIT_SHORT_SHA \
    SEEDPSD_COMMIT=$CI_COMMIT_SHORT_SHA \
    PATH="/app/.venv/bin:$PATH"

# For seedpsd-web
#ENTRYPOINT ["seedpsd-cli", "server", "web"]
# For seedpsd-amqp-data
#ENTRYPOINT ["seedpsd-cli", "server", "amqp-data"]
# For seedpsd-amqp-metadata
#ENTRYPOINT ["seedpsd-cli", "server", "amqp-metadata"]