# Squelette à compléter — étape 3.
# Mêmes contraintes que les autres services (multi-stage, non-root, ≤ 200 Mo).

# ---- Stage 1 : build / install des dépendances ----
FROM python:3.12.11-slim-bookworm AS build

WORKDIR /app

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app

# ---- Stage 2 : runtime ----
FROM python:3.12.11-slim-bookworm AS runtime

LABEL org.opencontainers.image.source="https://github.com/Yannis-Alouache/devhub-campus"

WORKDIR /app

COPY --from=build /opt/venv /opt/venv
COPY --from=build /app/app ./app

RUN useradd --create-home --uid 1001 app \
    && chown -R 1001:1001 /app /opt/venv

ENV PORT=8080 \
    LOG_LEVEL=info \
    PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:$PATH"

USER 1001:1001

EXPOSE 8080

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
