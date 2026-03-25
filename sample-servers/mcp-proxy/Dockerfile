# UWCU: Simplified Dockerfile for HTTP proxy mode only.
# Upstream installs Node.js + git + gnupg for stdio bridging (npx/uvx).
# We only use MCP_PROXY_URL (HTTP forwarding), so pure Python is sufficient.
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=1100:1100 . .

RUN adduser --disabled-password --gecos '' --uid 1100 mcpuser
USER mcpuser

EXPOSE 8000

CMD ["fastmcp", "run", "src/main.py:app", "--transport", "streamable-http", "--port", "8000", "--host", "0.0.0.0"]
