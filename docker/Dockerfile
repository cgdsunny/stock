# 使用阿里云镜像的基础镜像
FROM docker.io/python:3.11-slim-bullseye

LABEL maintainer="myh"

ENV LANG=zh_CN.UTF-8 \
    LC_CTYPE=zh_CN.UTF-8 \
    LC_ALL=C \
    PYTHONPATH=/data/InStock \
    C_INCLUDE_PATH=/usr/local/include/ta-lib \
    CPLUS_INCLUDE_PATH=/usr/local/include/ta-lib \
    LD_LIBRARY_PATH=/usr/local/lib

EXPOSE 9988

# 配置基础环境
RUN sed -i "s@http://$deb\|security$.debian.org@https://mirrors.aliyun.com@g" /etc/apt/sources.list && \
    printf "[global]\nindex-url = https://mirrors.aliyun.com/pypi/simple\ntrusted-host = mirrors.aliyun.com\n" > /etc/pip.conf && \
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone

# 安装系统依赖并编译TA-Lib
RUN apt-get update && \
    apt-get install -y \
        cron \
        gcc \
        make \
        python3-dev \
        default-libmysqlclient-dev \
        build-essential \
        pkg-config \
        curl \
        wget \
        libta-lib-dev && \
    # 编译安装TA-Lib
    mkdir -p /tmp/ta-lib && \
    wget -O /tmp/ta-lib.tar.gz https://downloads.sourceforge.net/project/ta-lib/ta-lib/0.4.0/ta-lib-0.4.0-src.tar.gz && \
    tar -xzf /tmp/ta-lib.tar.gz -C /tmp/ta-lib --strip-components=1 && \
    cd /tmp/ta-lib && \
    ./configure --prefix=/usr/local && \
    make -j$(nproc) && \
    make install && \
    cd / && \
    rm -rf /tmp/ta-lib* && \
    ldconfig && \  # 关键：更新动态链接库缓存
    # 安装Python依赖
    pip install --upgrade pip && \
    pip install TA-Lib --no-cache-dir && \
    # 清理构建工具
    apt-get purge -y \
        gcc \
        make \
        python3-dev \
        default-libmysqlclient-dev \
        build-essential \
        pkg-config \
        curl \
        wget && \
    apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /root/.cache/*

WORKDIR /data
COPY stock /data/InStock
COPY cron/cron.hourly /etc/cron.hourly
COPY cron/cron.workdayly /etc/cron.workdayly
COPY cron/cron.monthly /etc/cron.monthly

# 配置定时任务
RUN chmod 755 /data/InStock/instock/bin/run_*.sh && \
    chmod 755 /etc/cron.hourly/* /etc/cron.workdayly/* /etc/cron.monthly/* && \
    printf "SHELL=/bin/sh\nPATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin\n\n\
*/30 9,10,11,13,14,15 * * 1-5 /bin/run-parts /etc/cron.hourly\n\
30 17 * * 1-5 /bin/run-parts /etc/cron.workdayly\n\
30 10 * * 3,6 /bin/run-parts /etc/cron.monthly\n" > /var/spool/cron/crontabs/root && \
    chmod 600 /var/spool/cron/crontabs/root

ENTRYPOINT ["supervisord", "-n", "-c", "/data/InStock/supervisor/supervisord.conf"]


