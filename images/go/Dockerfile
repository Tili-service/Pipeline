# Use the official Golang image as base
FROM golang:1.17-alpine

# Set working directory
WORKDIR /app

# Copy the Go source code
COPY . .

# Build the application
RUN go mod init tili-backend && \
    go mod tidy && \
    go build -o server .

# Expose port 8081
EXPOSE 8081

# Command to run the server
CMD ["./server"]