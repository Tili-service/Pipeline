FROM golang:alpine as builder
WORKDIR /app
COPY . .
RUN go mod tidy
RUN go install github.com/swaggo/swag/cmd/swag@latest
RUN swag init --generalInfo cmd/api/main.go --output docs
RUN go build -o tili-backend cmd/api/main.go

FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/tili-backend .
EXPOSE 8080
CMD ["./tili-backend"]
