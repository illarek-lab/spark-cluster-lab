FROM eclipse-temurin:8-jre-jammy

USER root

ARG HADOOP_VERSION=3.3.4

RUN apt-get update \
    && apt-get install -y --no-install-recommends bash curl procps \
    && rm -rf /var/lib/apt/lists/*

ADD https://archive.apache.org/dist/hadoop/common/hadoop-${HADOOP_VERSION}/hadoop-${HADOOP_VERSION}.tar.gz /tmp/hadoop.tgz

RUN tar -xzf /tmp/hadoop.tgz -C /opt \
    && ln -s /opt/hadoop-${HADOOP_VERSION} /opt/hadoop \
    && rm /tmp/hadoop.tgz \
    && mkdir -p /opt/hadoop/logs /opt/hadoop/data/nameNode /opt/hadoop/data/dataNode \
    && chmod -R 777 /opt/hadoop/logs /opt/hadoop/data

COPY conf/hadoop/core-site.xml /opt/hadoop/etc/hadoop/core-site.xml
COPY conf/hadoop/hdfs-site.xml /opt/hadoop/etc/hadoop/hdfs-site.xml
COPY conf/hadoop/yarn-site.xml /opt/hadoop/etc/hadoop/yarn-site.xml
COPY conf/hadoop/mapred-site.xml /opt/hadoop/etc/hadoop/mapred-site.xml

ENV HADOOP_HOME=/opt/hadoop
ENV HADOOP_CONF_DIR=/opt/hadoop/etc/hadoop
ENV PATH=/opt/hadoop/bin:/opt/hadoop/sbin:$PATH

WORKDIR /opt/hadoop
