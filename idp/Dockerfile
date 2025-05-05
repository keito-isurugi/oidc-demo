FROM golang:1.24-alpine

WORKDIR /app
COPY . .

# airのインストール
RUN go install github.com/air-verse/air@latest

RUN go mod tidy

EXPOSE 8080

# airの実行コマンド
CMD ["air"]
