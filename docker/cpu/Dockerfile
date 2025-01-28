FROM --platform=linux/arm64 python:3.10-slim

# Install system dependencies - these are needed for our application
RUN apt-get update && apt-get install -y --no-install-recommends \
    espeak-ng \
    git \
    libsndfile1 \
    curl \
    ffmpeg \
    file \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user for security
RUN useradd -m -u 1000 appuser

# Create application directories with proper ownership
RUN mkdir -p /app/models && \
    mkdir -p /app/api/src/voices && \
    chown -R appuser:appuser /app

# Switch to non-root user for security
USER appuser
WORKDIR /app

# Download model files first - this helps with layer caching
WORKDIR /app/models
RUN set -x && \
    curl -L -o model.tar.gz https://github.com/remsky/Kokoro-FastAPI/releases/download/v0.0.1/kokoro-82m-onnx.tar.gz && \
    tar xzf model.tar.gz && \
    rm model.tar.gz

# Download voice models
WORKDIR /app/api/src/voices
RUN curl -L -o voices.tar.gz https://github.com/remsky/Kokoro-FastAPI/releases/download/v0.0.1/voice-models.tar.gz && \
    tar xzf voices.tar.gz && \
    rm voices.tar.gz

# Return to main application directory
WORKDIR /app

# Copy project configuration files
COPY --chown=appuser:appuser pyproject.toml ./pyproject.toml

# Install dependencies and set up virtual environment
# We'll use a different approach that doesn't rely on editable install
RUN python -m pip install --user pip uv && \
    echo "UV path: $(which uv)" && \
    $HOME/.local/bin/uv venv && \
    . .venv/bin/activate && \
    # Install dependencies from pyproject.toml without editable mode
    $HOME/.local/bin/uv pip install . && \
    # Install any additional dependencies your project needs
    $HOME/.local/bin/uv pip install \
        fastapi==0.109.2 \
        uvicorn==0.27.1 \
        python-multipart==0.0.7

# Copy application code
COPY --chown=appuser:appuser api ./api

# Set up environment variables
ENV PYTHONUNBUFFERED=1
ENV PYTHONPATH=/app:/app/models
ENV PATH="/app/.venv/bin:/home/appuser/.local/bin:$PATH"

# Run the FastAPI server
CMD ["python", "-m", "uvicorn", "api.src.main:app", "--host", "0.0.0.0", "--port", "8880", "--log-level", "debug"]