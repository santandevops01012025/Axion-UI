# syntax=docker/dockerfile:1

#############################################################
# Stage 1 - Resolve dependencies into a wheelhouse
#############################################################
FROM python:3.13-slim AS builder

WORKDIR /app

COPY requirements.txt .
RUN pip wheel --no-cache-dir -w /wheels -r requirements.txt

#############################################################
# Stage 2 - Runtime ships only wheels + the simulator
#############################################################
FROM python:3.13-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Non-root user
RUN addgroup --system --gid 1001 app && \
    adduser --system --uid 1001 --ingroup app app

WORKDIR /app

COPY --from=builder /wheels /wheels
RUN pip install --no-cache-dir --no-index --find-links=/wheels /wheels/* && rm -rf /wheels

COPY --chown=app:app simulator.py ./

USER app

CMD ["python", "-u", "simulator.py"]