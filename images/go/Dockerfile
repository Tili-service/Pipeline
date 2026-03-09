FROM golang:1.17-alpine

WORKDIR /app

SHELL ["/bin/sh", "-c"]

ONBUILD COPY app/go.mod /app/go.mod
ONBUILD COPY app/src /app/src/
ONBUILD RUN go mod download && \
    go build -o compiled-app /app/src/main.go && \
    mv compiled-app /app/compiled-app && \
    rm -rf src go.mod

ONBUILD CMD ["/app/compiled-app"]