# Single build for all three Go images. One shared builder stage compiles the three
# binaries, and each image is published from its own final stage. The bake targets in
# docker-bake.hcl pick a final stage through `target` and keep the image names unchanged.

FROM --platform=$BUILDPLATFORM quay.io/projectquay/golang:1.27 AS builder
ARG TARGETOS
ARG TARGETARCH

WORKDIR /workspace
# Copy the Go Modules manifests
COPY go.mod go.mod
COPY go.sum go.sum
# cache deps before building and copying source so that we don't need to re-download as much
# and so that source changes don't invalidate our downloaded layer
RUN go mod download

# Copy the go source
COPY cmd/ cmd/
COPY internal/ internal/
COPY pkg/ pkg/

# Build
# the GOARCH has not a default value to allow the binary be built according to the host where the command
# was called. For example, if we call make image-build in a local env which has the Apple Silicon M1 SO
# the docker BUILDPLATFORM arg will be linux/arm64 when for Apple x86 it will be linux/amd64. Therefore,
# by leaving it empty we can ensure that the container and binary shipped on it will have the same platform.
# Building every binary in one stage shares a single Go build cache, so internal/ and pkg/
# are compiled once instead of once per image.
# -o bin/ names each binary after its cmd/ directory: bin/apiserver, bin/batch-processor, bin/batch-gc.
RUN CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH} \
    go build -o bin/ ./cmd/apiserver ./cmd/batch-processor ./cmd/batch-gc

FROM gcr.io/distroless/static:nonroot AS apiserver
WORKDIR /
COPY --from=builder /workspace/bin/apiserver /app/batch-gateway-apiserver
USER 65532:65532

ENTRYPOINT ["/app/batch-gateway-apiserver"]

FROM gcr.io/distroless/static:nonroot AS processor
WORKDIR /
COPY --from=builder /workspace/bin/batch-processor /app/batch-gateway-processor
USER 65532:65532

ENTRYPOINT ["/app/batch-gateway-processor"]

FROM gcr.io/distroless/static:nonroot AS gc
WORKDIR /
COPY --from=builder /workspace/bin/batch-gc /app/batch-gateway-gc
USER 65532:65532

ENTRYPOINT ["/app/batch-gateway-gc"]
