FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY main.go .
RUN go build -o hello-app main.go

FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/hello-app .
EXPOSE 3000
CMD ["./hello-app"]
