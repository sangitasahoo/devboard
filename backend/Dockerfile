FROM golang:1.22-alpine

WORKDIR /app

COPY go.mod go.sum ./

RUN go mod download

# Build the binary.
COPY . .
RUN CGO_ENABLED=0 go build -o devboard-backend .

EXPOSE 8080
ENTRYPOINT ["./devboard-backend"]