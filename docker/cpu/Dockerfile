FROM --platform=linux/arm64 python:3.10-slim

# Install system dependencies
# These packages provide essential functionality for our speech processing application
RUN apt-get update && apt-get install -y --no-install-recommends \
    espeak-ng \
    git \
    libsndfile1 \
    curl \
    ffmpeg \
    file \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Create a non-root user for security
# This follows the principle of least privilege
RUN useradd -m -u 1000 appuser

# Set up application directories
# Creating these ahead of time ensures proper permissions
RUN mkdir -p /app/models && \
    mkdir -p /app/api/src/voices && \
    chown -R appuser:appuser /app

# Switch to non-root user
USER appuser
WORKDIR /app

# Download and extract model files
# We do this early in the build process to take advantage of Docker's layer caching
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

# Copy project configuration
COPY --chown=appuser:appuser pyproject.toml ./pyproject.toml

# Install dependencies without trying to install the project itself
# This approach avoids the package structure issues while ensuring all dependencies are available
RUN python -m pip install --user pip uv && \
    echo "UV path: $(which uv)" && \
    $HOME/.local/bin/uv venv && \
    . .venv/bin/activate && \
    # Install dependencies from pyproject.toml
    $HOME/.local/bin/uv pip install \
        fastapi==0.109.2 \
        uvicorn==0.27.1 \
        python-multipart==0.0.7 \
        pydantic==2.6.1 \
        numpy==1.26.3 \
        onnxruntime==1.16.3 \
        soundfile==0.12.1

# Copy application code
# We do this after installing dependencies to take advantage of Docker's layer caching
COPY --chown=appuser:appuser api ./api

# Set environment variables
# PYTHONPATH ensures Python can find our modules
# PYTHONUNBUFFERED ensures Python output isn't buffered, which is important for logging
ENV PYTHONUNBUFFERED=1
ENV PYTHONPATH=/app:/app/models
ENV PATH="/app/.venv/bin:/home/appuser/.local/bin:$PATH"

# Run the FastAPI server
CMD ["python", "-m", "uvicorn", "api.src.main:app", "--host", "0.0.0.0", "--port", "8880", "--log-level", "debug"]