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
COPY scripts/install-pulsar-client.sh /pulsar/bin

RUN mkdir /pulsar/data

### Create 2nd stage from Ubuntu image
### and add OpenJDK and Python dependencies (for Pulsar functions)

FROM ubuntu:20.04

ARG DEBIAN_FRONTEND=noninteractive


COPY auxiliary/sources-tuna-ubuntu2004.list /etc/apt/sources.list
RUN apt-get update && apt-get upgrade -y


# 编译python源码并安装
# 安装python先决条件【需要安装gcc make】
RUN apt-get install -y build-essential checkinstall
RUN apt-get install -y libncursesw5-dev libssl-dev \
    libsqlite3-dev tk-dev libgdbm-dev libc6-dev libbz2-dev libffi-dev zlib1g-dev gcc make
RUN apt-get clean all \
	&& mkdir /usr/local/python3

# 将下载好的python源码包copy到指定目录
COPY auxiliary/Python-3.7.8.tgz  /usr/local/python3/
# 编译python
RUN set -ex \
	&& cd /usr/local/python3 \
	&& tar -xvf  Python-3.7.8.tgz \
	&& cd Python-3.7.8 \
	&& ./configure prefix=/usr/local/python3 \
	&& make && make install \
	&& ln -s /usr/local/python3/bin/python3.7 /usr/local/bin/python3 \
	&& ln -s /usr/local/python3/bin/pip3 /usr/local/bin/pip3 \
    && ln -s /usr/local/python3/bin/python3.7 /usr/bin/python3 \
    && ln -s /usr/local/python3/bin/pip3 /usr/bin/pip3
RUN set -ex \
	&& cd /usr/local/python3 \
	&& python3 -V


# Install some utilities
RUN apt-get update \
     && apt-get -y dist-upgrade \
     && apt-get -y install openjdk-11-jdk-headless  netcat dnsutils less procps iputils-ping \
                 python3-setuptools python3-yaml python3-kazoo \
                 libreadline-gplv2-dev libncursesw5-dev libssl-dev libsqlite3-dev tk-dev libgdbm-dev libc6-dev libbz2-dev \
                 curl \
     && apt-get -y --purge autoremove \
     && apt-get autoclean \
     && apt-get clean \
     && rm -rf /var/lib/apt/lists/*

RUN curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
RUN python3 get-pip.py

RUN update-alternatives --install /usr/bin/python python /usr/bin/python3 10

ENV JAVA_HOME /usr/lib/jvm/java-11-openjdk-amd64
RUN echo networkaddress.cache.ttl=1 >> /usr/lib/jvm/java-11-openjdk-amd64/conf/security/java.security
ADD target/python-client/ /pulsar/pulsar-client

ENV PULSAR_ROOT_LOGGER=INFO,CONSOLE


COPY --from=pulsar /pulsar /pulsar
WORKDIR /pulsar

RUN /pulsar/bin/install-pulsar-client.sh