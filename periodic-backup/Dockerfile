FROM alpine:3.16.3

COPY ./scripts/daily/* /etc/periodic/daily

# 時刻をJSTにする。MySQL接続用クライアントをインストールする
RUN apk update && \
    apk upgrade && \
    apk --no-cache add tzdata && \
    cp /usr/share/zoneinfo/Asia/Tokyo /etc/localtime && \
    apk del tzdata && \
    apk add --no-cache mysql-client && \
    chmod a+x /etc/periodic/daily/*
