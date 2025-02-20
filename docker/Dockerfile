# 使用阿里云镜像的基础镜像
FROM docker.io/python:3.11-slim-bullseye

MAINTAINER myh
ENV LANG=zh_CN.UTF-8 \
    LC_CTYPE=zh_CN.UTF-8 \
    LC_ALL=C \
    PYTHONPATH=/data/InStock \
    # 关键：强制指定头文件和库路径
    C_INCLUDE_PATH=/usr/local/include/ta-lib/ \
    CPLUS_INCLUDE_PATH=/usr/local/include/ta-lib/ \
    LD_LIBRARY_PATH=/usr/local/lib

EXPOSE 9988

# 配置阿里云镜像源、时区和依赖安装
RUN sed -i "s@http://$deb\|security$.debian.org@https://mirrors.aliyun.com@g" /etc/apt/sources.list && \
    printf "[global]\nindex-url = https://mirrors.aliyun.com/pypi/simple\ntrusted-host = mirrors.aliyun.com\n" > /etc/pip.conf && \
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone && \
    # 安装系统依赖（包含 TA-Lib 编译所需）
    apt-get update && \
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
        # 关键：安装 TA-Lib 开发头文件
        libta-lib-dev \
        && \
    # 手动编译 TA-Lib（覆盖系统库）
    mkdir -p /tmp/ta-lib && \
    wget -O /tmp/ta-lib.tar.gz https://downloads.sourceforge.net/project/ta-lib/ta-lib/0.4.0/ta-lib-0.4.0-src.tar.gz && \
    tar -xzf /tmp/ta-lib.tar.gz -C /tmp/ta-lib --strip-components=1 && \
    cd /tmp/ta-lib && \
    ./configure --prefix=/usr/local \  # 关键：安装到 /usr/local
    && make -j$(nproc) \
    && make install \
    && cd / \
    && rm -rf /tmp/ta-lib* \
    # 验证头文件路径（调试用）
    && echo "--- 头文件验证 ---" \
    && ls -l /usr/local/include/ta-lib/ta_defs.h \
    && echo "--- 库文件验证 ---" \
    && ldconfig -p | grep ta_lib \
    # 升级 pip 并安装 Python 依赖
    && pip install --upgrade pip \
    && pip install \
        TA-Lib \  # 注意包名是短横线
        --no-cache-dir \
    # 清理构建工具（确保在最后一步清理）
    && apt-get purge -y \
        gcc \
        make \
        python3-dev \
        default-libmysqlclient-dev \
        build-essential \
        pkg-config \
        curl \
        wget \
    && apt-get autoremove -y \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /root/.cache/*

# 其他步骤保持不变...

WORKDIR /data
#InStock软件
COPY stock /data/InStock
COPY cron/cron.hourly /etc/cron.hourly
COPY cron/cron.workdayly /etc/cron.workdayly
COPY cron/cron.monthly /etc/cron.monthly

#add cron sesrvice.
#任务调度
RUN chmod 755 /data/InStock/instock/bin/run_*.sh && \
    chmod 755 /etc/cron.hourly/* && chmod 755 /etc/cron.workdayly/* && chmod 755 /etc/cron.monthly/* && \
    echo "SHELL=/bin/sh \n\
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin \n\
# min hour day month weekday command \n\
*/30 9,10,11,13,14,15 * * 1-5 /bin/run-parts /etc/cron.hourly \n\
30 17 * * 1-5 /bin/run-parts /etc/cron.workdayly \n\
30 10 * * 3,6 /bin/run-parts /etc/cron.monthly \n" > /var/spool/cron/crontabs/root && \
    chmod 600 /var/spool/cron/crontabs/root

ENTRYPOINT ["supervisord","-n","-c","/data/InStock/supervisor/supervisord.conf"]

# 检查编译器搜索路径
RUN echo "--- 编译器包含路径 ---" \
    && gcc -xc -E -v - 2>&1 | sed -n '/#include <...>/,/End of search list/p'
