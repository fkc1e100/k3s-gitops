FROM python:3.13-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    openssh-client \
    && rm -rf /var/lib/apt/lists/*

# Install hermes-agent and core requirements
RUN pip install --no-cache-dir \
    hermes-agent==0.15.2 \
    discord.py==2.7.1 \
    python-dotenv==1.2.2 \
    PyYAML==6.0.3 \
    requests==2.33.0 \
    psutil==7.2.2 \
    croniter==6.0.0

# Set up environment variables
ENV HERMES_HOME=/workspace/.hermes
ENV VIRTUAL_ENV=/workspace/venv
ENV PATH="/workspace/venv/bin:$PATH"

WORKDIR /workspace

# Run the gateway by default
ENTRYPOINT ["python", "-m", "hermes_cli.main", "gateway", "run"]
