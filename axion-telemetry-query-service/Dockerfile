# syntax=docker/dockerfile:1

#############################################################
# Stage 1 - Resolve dependencies into a wheelhouse
#############################################################
FROM python:3.13-slim-bookworm AS builder

WORKDIR /app

COPY requirements.txt .
RUN pip wheel --no-cache-dir -w /wheels -r requirements.txt

#############################################################
# Stage 2 - Runtime only ships wheels + code (no build tooling)
#############################################################
FROM python:3.13-slim-bookworm AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/usr/local/bin"

WORKDIR /app

# Install prebuilt wheels from the builder stage, then drop them
COPY --from=builder /wheels /wheels
RUN pip install --no-cache-dir --no-index --find-links=/wheels /wheels/*

COPY --chown=1001:1001 main.py config.py database.py ./

USER 1001

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=3)" || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]