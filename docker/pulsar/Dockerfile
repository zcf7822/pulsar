#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#

# First create a stage with just the Pulsar tarball and scripts
FROM busybox as pulsar

ARG PULSAR_TARBALL

ADD ${PULSAR_TARBALL} /
RUN mv /apache-pulsar-* /pulsar

COPY scripts/apply-config-from-env.py /pulsar/bin
COPY scripts/apply-config-from-env-with-prefix.py /pulsar/bin
COPY scripts/gen-yml-from-env.py /pulsar/bin
COPY scripts/generate-zookeeper-config.sh /pulsar/bin
COPY scripts/pulsar-zookeeper-ruok.sh /pulsar/bin
COPY scripts/watch-znode.py /pulsar/bin
COPY scripts/install-pulsar-client-37.sh /pulsar/bin

RUN mkdir /pulsar/data

### Create 2nd stage from OpenJDK image
### and add Python dependencies (for Pulsar functions)

FROM openjdk:11-jdk-slim

# 更改国内源时添加签名
#RUN apt-get update && apt-get install -y gnupg2 \
#    && apt-key adv --keyserver keyserver.ubuntu.com --recv-keys  7638D0442B90D010 \
#    && apt-key adv --keyserver keyserver.ubuntu.com --recv-keys  CBF8D6FD518E17E1

# 更改国内源
COPY ../auxiliary/sources-tuna-bullseye.list /etc/apt/sources.list
RUN apt-get update && apt-get upgrade

# 清华源安装https证书
RUN apt install -y apt-transport-https ca-certificates

# 编译python源码并安装
# 安装python先决条件【需要安装gcc make】
RUN apt-get install -y build-essential checkinstall
RUN apt-get install -y libncursesw5-dev libssl-dev \
    libsqlite3-dev tk-dev libgdbm-dev libc6-dev libbz2-dev libffi-dev zlib1g-dev gcc make
RUN apt-get clean all \
	&& mkdir /usr/local/python3

# 将下载好的python源码包copy到指定目录
COPY ../auxiliary/Python-3.7.8.tgz  /usr/local/python3/
# 编译python
RUN set -ex \
	&& cd /usr/local/python3 \
	&& tar -xvf  Python-3.7.8.tgz \
	&& cd Python-3.7.8 \
	&& ./configure prefix=/usr/local/python3 \
	&& make && make install \
	&& ln -s /usr/local/python3/bin/python3.7 /usr/local/bin/python3.7 \
	&& ln -s /usr/local/python3/bin/pip3 /usr/local/bin/pip3.7 \
    && ln -s /usr/local/python3/bin/python3.7 /usr/bin/python3.7 \
    && ln -s /usr/local/python3/bin/pip3 /usr/bin/pip3.7
RUN set -ex \
	&& cd /usr/local/python3 \
	&& python3.7 -V


# Install some utilities
RUN apt-get update && apt-get install -y netcat dnsutils less procps iputils-ping \
                 python3-setuptools python3-yaml python3-kazoo \
                 libreadline-gplv2-dev libncursesw5-dev libssl-dev libsqlite3-dev tk-dev libgdbm-dev libc6-dev libbz2-dev \
                 curl \
     && apt-get clean \
     && rm -rf /var/lib/apt/lists/*
#
RUN curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
RUN python3.7 get-pip.py

RUN update-alternatives --install /usr/bin/python python /usr/bin/python3 10

RUN echo networkaddress.cache.ttl=1 >> $JAVA_HOME/conf/security/java.security
ADD target/python-client/ /pulsar/pulsar-client

VOLUME  ["/pulsar/conf", "/pulsar/data"]

ENV PULSAR_ROOT_LOGGER=INFO,CONSOLE


COPY --from=pulsar /pulsar /pulsar
WORKDIR /pulsar

RUN /pulsar/bin/install-pulsar-client-37.sh

