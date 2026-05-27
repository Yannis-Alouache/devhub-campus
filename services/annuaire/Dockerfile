# Squelette à compléter — étape 3 du TP.
# Contraintes (non négociables, lisez le polycopié) :
#   - multi-stage,
#   - image finale ≤ 200 Mo,
#   - utilisateur non-root,
#   - pas de secret en ENV/ARG,
#   - LABEL org.opencontainers.image.source pointant vers le dépôt Git.

# ---- Stage 1 : build / install des dépendances ----
# Image pinnee sur une version Node/Alpine precise pour garder un build reproductible.
FROM node:20.19.5-alpine3.22 AS build

WORKDIR /app

COPY .npmrc package*.json ./
RUN npm ci --omit=dev

COPY src ./src

# ---- Stage 2 : image runtime ----
FROM node:20.19.5-alpine3.22 AS runtime

LABEL org.opencontainers.image.source="https://github.com/Yannis-Alouache/devhub-campus"

WORKDIR /app

COPY --from=build /app/package.json ./package.json
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/src ./src

RUN addgroup -S app -g 1001 \
    && adduser -S app -G app -u 1001 \
    && chown -R 1001:1001 /app

ENV PORT=8080 \
    LOG_LEVEL=info \
    NODE_ENV=production

USER 1001:1001

EXPOSE 8080

CMD ["node", "src/index.js"]
