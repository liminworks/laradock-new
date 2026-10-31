# hyperf 运行时基础镜像（本地开发用）
#
# api/im 共用（tag hyperf:dev）；代码由 compose 运行时挂载到 /opt/www，
# 镜像不含代码与 vendor（宿主机项目内需已 composer install）。
#
# @link     https://www.hyperf.io
# @document https://doc.hyperf.io

FROM hyperf/hyperf:8.1-alpine-v3.18-swoole
LABEL maintainer="limin" version="1.0" license="MIT" app.name="qwr-hyperf"

##
# ---------- env settings ----------
##
# --build-arg timezone=Asia/Shanghai
ARG timezone

ENV TIMEZONE=${timezone:-"Asia/Shanghai"} \
    APP_ENV=test \
    SCAN_CACHEABLE=(true)

# update
RUN set -ex \
    # alpine 官方 CDN 大陆直连极慢，换阿里云镜像源（境外构建不受影响）
    && sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories \
    && apk update \
    # install ffmpeg binary (audio convert, called via Swoole\Coroutine::exec)
    && apk add --no-cache ffmpeg \
    # composer 2.x 已内置于基础镜像，无需从 github 下载
    # show php version and extensions
    && php -v \
    && php -m \
    && php --ri swoole \
    #  ---------- some config ----------
    && cd /etc/php81 \
    # - config PHP
    && { \
        echo "upload_max_filesize=100M"; \
        echo "post_max_size=108M"; \
        echo "memory_limit=1024M"; \
        echo "date.timezone=${TIMEZONE}"; \
    } | tee conf.d/99_overrides.ini \
    # - config timezone
    && ln -sf /usr/share/zoneinfo/${TIMEZONE} /etc/localtime \
    && echo "${TIMEZONE}" > /etc/timezone \
    # ---------- clear works ----------
    && rm -rf /var/cache/apk/* /tmp/* /usr/share/man \
    && echo -e "\033[42;37m Build Completed :).\033[0m\n"

WORKDIR /opt/www
