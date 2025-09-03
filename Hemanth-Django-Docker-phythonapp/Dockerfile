### Stage 1: Builder
FROM python:3.11-slim AS builder

WORKDIR /app

RUN apt-get update && apt-get install -y \
    build-essential gcc curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

### Stage 2: Final
FROM python:3.11-slim

WORKDIR /app

# Copy installed dependencies
COPY --from=builder /install /usr/local

# Copy project
COPY Hemanth /app/Hemanth

# Environment
ENV DJANGO_SETTINGS_MODULE=Hemanth.settings
ENV PYTHONUNBUFFERED=1
ENV PYTHONPATH=/app/Hemanth

EXPOSE 8000

# Ensure staticfiles folder exists and run migrations/collectstatic before Gunicorn
CMD mkdir -p /app/Hemanth/staticfiles && \
    python3 Hemanth/manage.py migrate --noinput && \
    python3 Hemanth/manage.py collectstatic --noinput && \
    gunicorn --chdir /app/Hemanth Hemanth.wsgi:application --bind 0.0.0.0:8000 --workers 4

