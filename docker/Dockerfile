# syntax=docker/dockerfile:1.7
###############################################################################
# Stage 1: builder —— 交叉编译 gobgpd（守护进程）与 gobgp（CLI）
#
#   1) FROM --platform=$BUILDPLATFORM：重活（go build）始终在构建机原生架构上
#      完成，TARGETARCH 仅决定交叉编译目标，所以多架构构建无需在 builder 里跑
#      QEMU。
#   2) 版本信息通过 ldflags 注入 internal/pkg/version 包（见 CONTRIBUTING.md），
#      与上游一致：COMMIT=<git sha>、METADATA=date.<YYYYMMDD>。gobgp 没有
#      hack/get_version.sh / .version 那一套，由 CI 通过 build-arg 传入即可。
#   3) CGO_ENABLED=0 + -tags netgo + -a，与 .goreleaser.yml 一致，产物是纯静态
#      二进制，runtime 不需要 glibc。
###############################################################################
FROM --platform=$BUILDPLATFORM golang:1.25-alpine3.23 AS builder

# CI 传入：VERSION 为人类可读标签（短 sha 或 tag），COMMIT 为 git 短 sha，
# DATE 为构建日期（YYYYMMDD），都用于 ldflags / OCI label。
ARG VERSION="dev"
ARG COMMIT=""
ARG DATE=""

# 兼容本地 `docker build --build-arg goarch=...` 的用法
ARG goarch=

ARG TARGETOS
ARG TARGETARCH
ARG BUILDOS
ARG BUILDARCH

ENV GOOS=${TARGETOS:-linux} \
    GOARCH=${goarch:-${TARGETARCH:-amd64}} \
    CGO_ENABLED=0 \
    GOFLAGS=-mod=mod

WORKDIR /workdir

# 先拷依赖清单并预热模块缓存，源码改动时这一层可命中缓存。
COPY go.mod go.sum ./
RUN go mod download

COPY . .

# -X 路径必须与模块路径一致（go.mod: github.com/osrg/gobgp/v4）。
RUN set -eux; \
    PKG=github.com/osrg/gobgp/v4/internal/pkg/version; \
    LDFLAGS="-s -w"; \
    if [ -n "$COMMIT" ]; then LDFLAGS="$LDFLAGS -X ${PKG}.COMMIT=${COMMIT}"; fi; \
    if [ -n "$DATE" ];   then LDFLAGS="$LDFLAGS -X ${PKG}.METADATA=date.${DATE}"; fi; \
    mkdir -p /workdir/bin; \
    go build -a -tags netgo -ldflags="$LDFLAGS" -o /workdir/bin/gobgpd ./cmd/gobgpd; \
    go build -a -tags netgo -ldflags="$LDFLAGS" -o /workdir/bin/gobgp  ./cmd/gobgp

###############################################################################
# Stage 2: runtime —— 仅保留 gobgpd / gobgp 运行所需
#
#   - gobgpd 是常驻守护进程：监听 BGP(179) 与 gRPC API(:50051)，所以设置
#     ENTRYPOINT。gobgp CLI 一并装入 PATH，便于 `docker exec ... gobgp ...`。
#   - ca-certificates 供 gRPC TLS / Sentry 等出站 TLS 校验使用。
###############################################################################
FROM alpine:3.23

ARG VERSION="dev"

LABEL org.opencontainers.image.title="gobgp" \
      org.opencontainers.image.description="GoBGP — BGP implementation in Go (Alpine build): gobgpd daemon + gobgp CLI" \
      org.opencontainers.image.source="https://github.com/fivetime/gobgp" \
      org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.licenses="Apache-2.0"

RUN apk add --no-cache ca-certificates

COPY --from=builder /workdir/bin/gobgpd /usr/local/bin/gobgpd
COPY --from=builder /workdir/bin/gobgp  /usr/local/bin/gobgp

# 179   : BGP
# 50051 : gobgpd gRPC API（默认 --api-hosts=:50051）
EXPOSE 179/tcp 50051/tcp

ENTRYPOINT ["gobgpd"]
# 默认无参启动：gobgpd 不带 -f 时会等待通过 gRPC 下发配置。
# 如需文件配置：docker run ... gobgp:latest -f /etc/gobgp/gobgpd.toml
CMD []
