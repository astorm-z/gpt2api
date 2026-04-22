# syntax=docker/dockerfile:1.7

ARG GO_VERSION=1.26.1
ARG NODE_VERSION=20

FROM --platform=$BUILDPLATFORM golang:${GO_VERSION}-alpine AS goose-builder
ARG TARGETOS
ARG TARGETARCH

RUN apk add --no-cache ca-certificates git

ENV CGO_ENABLED=0
WORKDIR /src

RUN mkdir -p /out \
    && GOBIN=/out GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    go install github.com/pressly/goose/v3/cmd/goose@v3.20.0

FROM --platform=$BUILDPLATFORM golang:${GO_VERSION}-alpine AS app-builder
ARG TARGETOS
ARG TARGETARCH

RUN apk add --no-cache ca-certificates git

WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download

COPY cmd ./cmd
COPY internal ./internal
COPY pkg ./pkg

RUN GOOS=${TARGETOS} GOARCH=${TARGETARCH} CGO_ENABLED=0 \
    go build -trimpath -ldflags "-s -w" -o /out/gpt2api ./cmd/server

FROM --platform=$BUILDPLATFORM node:${NODE_VERSION}-alpine AS web-builder
WORKDIR /src/web

COPY web/package.json web/package-lock.json ./
RUN npm ci --no-audit --no-fund

COPY web/ ./
RUN npm run build

FROM alpine:3.20

RUN apk add --no-cache ca-certificates tzdata curl bash mariadb-client \
    && update-ca-certificates \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone

WORKDIR /app

COPY --from=app-builder /out/gpt2api /app/gpt2api
COPY --from=goose-builder /out/goose /usr/local/bin/goose
RUN chmod +x /app/gpt2api /usr/local/bin/goose

COPY --from=web-builder /src/web/dist /app/web/dist

COPY sql /app/sql
COPY configs /app/configs
COPY deploy/entrypoint.sh /app/entrypoint.sh
RUN sed -i 's/\r$//' /app/entrypoint.sh \
    && chmod +x /app/entrypoint.sh \
    && mkdir -p /app/data/backups /app/logs

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
    CMD curl -fsS http://localhost:8080/healthz || exit 1

ENTRYPOINT ["/app/entrypoint.sh"]
CMD ["/app/gpt2api", "-c", "/app/configs/config.yaml"]
