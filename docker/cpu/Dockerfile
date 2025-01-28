FROM --platform=linux/arm64 python:3.10-slim

# Install all required packages in a single RUN command to minimize layers
# Each package installation is joined with && and line continuations (\)
RUN apt-get update && apt-get install -y --no-install-recommends \
    espeak-ng \
    git \
    libsndfile1 \
    curl \
    ffmpeg \
    file \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Run diagnostic commands to verify our architecture
# All commands are part of a single RUN instruction
RUN echo "Architecture details:" && \
    uname -a && \
    file /bin/ls && \
    dpkg --print-architecture

# Install uv with verification steps
# Using a single RUN command for the entire installation process
RUN mkdir -p /tmp/uv && \
    cd /tmp/uv && \
    echo "Downloading uv for ARM64..." && \
    curl -L -o uv.tar.gz https://github.com/astral-sh/uv/releases/download/0.5.24/uv-aarch64-unknown-linux-gnu.tar.gz && \
    tar xzf uv.tar.gz --strip-components=1 && \
    echo "Examining uv binary:" && \
    file uv && \
    mv uv /usr/local/bin/ && \
    chmod +x /usr/local/bin/uv && \
    cd / && \
    rm -rf /tmp/uv

# Create non-root user
RUN useradd -m -u 1000 appuser

# Create directories and set ownership
RUN mkdir -p /app/models && \
    mkdir -p /app/api/src/voices && \
    chown -R appuser:appuser /app

USER appuser

# Download and extract models
WORKDIR /app/models
RUN set -x && \
    curl -L -o model.tar.gz https://github.com/remsky/Kokoro-FastAPI/releases/download/v0.0.1/kokoro-82m-onnx.tar.gz && \
    tar xzf model.tar.gz && \
    rm model.tar.gz

# Download and extract voice models
WORKDIR /app/api/src/voices
RUN curl -L -o voices.tar.gz https://github.com/remsky/Kokoro-FastAPI/releases/download/v0.0.1/voice-models.tar.gz && \
    tar xzf voices.tar.gz && \
    rm voices.tar.gz

# Switch back to app directory
WORKDIR /app

# Copy dependency files
COPY --chown=appuser:appuser pyproject.toml ./pyproject.toml

# Install dependencies using pip for initial setup
# Each command is connected with && and uses line continuations
RUN python -m pip install --user pip uv && \
    echo "UV path: $(which uv)" && \
    $HOME/.local/bin/uv venv && \
    . .venv/bin/activate && \
    $HOME/.local/bin/uv pip install --no-deps -r requirements.txt

# Copy project files
COPY --chown=appuser:appuser api ./api

# Install project
RUN . .venv/bin/activate && \
    $HOME/.local/bin/uv pip install -e .

# Set environment variables
ENV PYTHONUNBUFFERED=1
ENV PYTHONPATH=/app:/app/models
ENV PATH="/app/.venv/bin:/home/appuser/.local/bin:$PATH"

# Run FastAPI server
CMD ["python", "-m", "uvicorn", "api.src.main:app", "--host", "0.0.0.0", "--port", "8880", "--log-level", "debug"]