FROM golang:1.24-alpine

WORKDIR /app
COPY . .

RUN go install github.com/air-verse/air@latest
RUN go mod tidy

EXPOSE 8081

CMD ["air"]
