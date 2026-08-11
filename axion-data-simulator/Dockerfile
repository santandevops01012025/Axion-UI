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

WORKDIR /app

COPY --from=builder /wheels /wheels
RUN pip install --no-cache-dir --no-index --find-links=/wheels /wheels/* && rm -rf /wheels

COPY --chown=1001:1001 simulator.py ./

USER 1001

CMD ["python", "-u", "simulator.py"]