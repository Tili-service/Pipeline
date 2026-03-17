FROM golang:1.24-alpine

WORKDIR /app

SHELL ["/bin/sh", "-c"]

ONBUILD COPY . .

ONBUILD RUN go mod tidy
ONBUILD RUN go install github.com/swaggo/swag/cmd/swag@latest
ONBUILD RUN swag init --generalInfo cmd/main.go --output docs
ONBUILD RUN go build -o tili-server ./cmd

ONBUILD EXPOSE 8000

ONBUILD CMD ["./tili-server"]
