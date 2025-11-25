FROM python:3.10-slim 

# Install system dependencies FIRST (before switching user)
RUN apt-get update -y &&  \
    apt-get install -y espeak-ng espeak-ng-data git libsndfile1 curl ffmpeg g++ build-essential && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    mkdir -p /usr/share/espeak-ng-data && \
    ln -s /usr/lib/*/espeak-ng-data/* /usr/share/espeak-ng-data/ 2>/dev/null || true && \
    curl -LsSf https://astral.sh/uv/install.sh | sh && \
    mv /root/.local/bin/uv /usr/local/bin/ && \
    mv /root/.local/bin/uvx /usr/local/bin/

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    mkdir -p /app/api/src/models/v1_0 && \
    chown -R appuser:appuser /app

USER appuser
WORKDIR /app

# Install Rust for the non-root user (needed for sudachipy, pyopenjtalk-plus, etc.)
RUN curl https://sh.rustup.rs -sSf | sh -s -- -y

# Set environment variables for build and runtime
ENV PATH="/home/appuser/.cargo/bin:/app/.venv/bin:$PATH" \
    UV_HTTP_TIMEOUT=120 \
    UV_HTTP_RETRIES=3 \
    RUSTFLAGS="-C target-feature=+crt-static" \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app:/app/api \
    UV_LINK_MODE=copy \
    USE_GPU=false \
    PHONEMIZER_ESPEAK_PATH=/usr/bin \
    PHONEMIZER_ESPEAK_DATA=/usr/share/espeak-ng-data \
    ESPEAK_DATA_PATH=/usr/share/espeak-ng-data \
    DEVICE="cpu"

# Copy dependency files
COPY --chown=appuser:appuser pyproject.toml ./pyproject.toml

# Create venv and install dependencies
# Note: Rust must be in PATH before this runs, hence the single RUN command
RUN . "$HOME/.cargo/env" && \
    uv venv --python 3.10 && \
    . .venv/bin/activate && \
    uv sync --extra cpu --no-cache

# Copy project files including models
COPY --chown=appuser:appuser api ./api
COPY --chown=appuser:appuser web ./web
COPY --chown=appuser:appuser docker/scripts/ ./
RUN chmod +x ./entrypoint.sh

# Download spacy model after dependencies are installed
RUN . .venv/bin/activate && \
    python -m spacy download en_core_web_sm 2>/dev/null || true

ENV DOWNLOAD_MODEL=true
# Download model if enabled
RUN if [ "$DOWNLOAD_MODEL" = "true" ]; then \
    . .venv/bin/activate && \
    python download_model.py --output api/src/models/v1_0; \
    fi

# Run FastAPI server through entrypoint.sh
CMD ["./entrypoint.sh"]