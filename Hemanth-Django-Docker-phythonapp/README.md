Stage 1: Builder
FROM python:3.11-slim AS builder
WORKDIR /app


FROM python:3.11-slim AS builder:
Starts a builder stage using a lightweight Python 3.11 image.

This stage is used to install all dependencies without bloating the final image.

WORKDIR /app:
Sets /app as the working directory inside the container.
Any subsequent COPY or RUN commands will operate relative to this folder.

RUN apt-get update && apt-get install -y \
    build-essential gcc curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*


Updates the package index and installs build tools required to compile some Python packages (build-essential, gcc) and system utilities (curl, ca-certificates).

rm -rf /var/lib/apt/lists/* cleans up cached package lists to reduce image size.

COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt


COPY requirements.txt .: copies the requirements.txt file into /app.

RUN pip install --no-cache-dir --prefix=/install -r requirements.txt:

Installs all Python dependencies into /install, not the system site-packages.

This allows us to copy them cleanly into the final image without including unnecessary build tools.

Stage 2: Final
FROM python:3.11-slim
WORKDIR /app


Starts the final runtime image, again based on python:3.11-slim.

WORKDIR /app ensures that all operations happen relative to /app.

# Copy installed dependencies
COPY --from=builder /install /usr/local


Copies Python packages and executables from the builder stage to /usr/local in the final image.

This includes all installed libraries like Django, Gunicorn, etc., without including build tools.

# Copy project
COPY Hemanth /app/Hemanth


Copies your Django project folder Hemanth into /app/Hemanth in the container.

This ensures that manage.py and your inner project files (Hemanth/settings.py, Hemanth/wsgi.py) are present in the container.

# Environment
ENV DJANGO_SETTINGS_MODULE=Hemanth.settings
ENV PYTHONUNBUFFERED=1
ENV PYTHONPATH=/app/Hemanth


DJANGO_SETTINGS_MODULE: tells Django which settings file to use.

PYTHONUNBUFFERED=1: ensures Python output goes straight to the console (useful for logs).

PYTHONPATH=/app/Hemanth: allows Python to locate your project modules correctly inside the container.

EXPOSE 8000


Tells Docker that the container listens on port 8000.

This is the port Django/Gunicorn will serve traffic on.

# Ensure staticfiles folder exists and run migrations/collectstatic before Gunicorn
CMD mkdir -p /app/Hemanth/staticfiles && \
    python3 Hemanth/manage.py migrate --noinput && \
    python3 Hemanth/manage.py collectstatic --noinput && \
    gunicorn --chdir /app/Hemanth Hemanth.wsgi:application --bind 0.0.0.0:8000 --workers 4


This is a multi-step CMD, which does the following every time the container starts:

mkdir -p /app/Hemanth/staticfiles

Ensures the folder for static files exists so collectstatic can copy files there.

python3 Hemanth/manage.py migrate --noinput

Applies any pending Django database migrations automatically.

--noinput skips interactive prompts.

python3 Hemanth/manage.py collectstatic --noinput

Collects all static files (CSS, JS, images) into the /app/Hemanth/staticfiles folder for serving.

gunicorn --chdir /app/Hemanth Hemanth.wsgi:application --bind 0.0.0.0:8000 --workers 4

Starts Gunicorn, the production WSGI server.

--chdir /app/Hemanth ensures Gunicorn runs in the correct working directory where manage.py and the project are.

Hemanth.wsgi:application points to the WSGI application.

--workers 4 runs 4 worker processes to handle requests.

✅ Summary

Builder stage: installs Python dependencies cleanly without bloating final image.

Final stage: runtime-only image with your Django project + dependencies.

CMD: prepares the environment (static files, migrations) and starts Gunicorn in production mode.
