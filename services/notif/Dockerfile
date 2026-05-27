# Squelette — étape 3.
# Cible : image FINALE ≤ 50 Mo (Go produit un binaire statique, profitez-en).

# ---- Stage 1 : build du binaire Go ----
FROM golang:1.22.12-alpine3.21 AS build

WORKDIR /src

COPY go.mod ./
RUN go mod download

COPY cmd ./cmd
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /out/notif ./cmd

# ---- Stage 2 : runtime minimal ----
FROM gcr.io/distroless/static:nonroot AS runtime

LABEL org.opencontainers.image.source="https://github.com/Yannis-Alouache/devhub-campus"

COPY --from=build /out/notif /notif

ENV PORT=8080 \
    LOG_LEVEL=info

EXPOSE 8080

# USER nonroot:nonroot     # déjà fourni par l'image distroless
ENTRYPOINT ["/notif"]
