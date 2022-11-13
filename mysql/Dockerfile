# redmineで指定のバージョン(aptで使用するのでdebianを指定、デフォルトはoracle linuxらしい)
FROM  mysql:5.7-debian

# vimとpsコマンドをインストール
RUN apt-get update && apt-get install -y \
    procps \
    vim
# cron \
# supervisor

# ローカルに用意したcron設定ファイル、supervisord設定ファイルをコピー
# COPY ./cron/cron.d /etc/cron.d/
# RUN chmod 0644 /etc/cron.d/*

# COPY ./supervisor/supervisord.conf /etc/supervisor/

# # supervisor起動
# CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisord.conf"]
