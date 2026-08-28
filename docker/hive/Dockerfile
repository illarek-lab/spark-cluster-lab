FROM eclipse-temurin:8-jre-jammy

USER root

ARG HIVE_VERSION=3.1.0
ARG POSTGRES_VERSION=42.7.4
ARG HADOOP_VERSION=3.3.4
ARG HADOOP_AWS_VERSION=3.3.4
ARG AWS_SDK_BUNDLE_VERSION=1.12.262

RUN apt-get update \
    && apt-get install -y --no-install-recommends bash curl procps \
    && rm -rf /var/lib/apt/lists/*

ADD https://archive.apache.org/dist/hadoop/common/hadoop-${HADOOP_VERSION}/hadoop-${HADOOP_VERSION}.tar.gz /tmp/hadoop.tgz

RUN tar -xzf /tmp/hadoop.tgz -C /opt \
    && ln -s /opt/hadoop-${HADOOP_VERSION} /opt/hadoop \
    && rm /tmp/hadoop.tgz

ADD https://archive.apache.org/dist/hive/hive-${HIVE_VERSION}/apache-hive-${HIVE_VERSION}-bin.tar.gz /tmp/hive.tgz

RUN tar -xzf /tmp/hive.tgz -C /opt \
    && ln -s /opt/apache-hive-${HIVE_VERSION}-bin /opt/hive \
    && rm /tmp/hive.tgz \
    && mkdir -p /opt/hive/logs /opt/hive/warehouse /tmp/hive \
    && chmod -R 777 /opt/hive/logs /opt/hive/warehouse /tmp/hive

ADD https://repo.maven.apache.org/maven2/org/postgresql/postgresql/${POSTGRES_VERSION}/postgresql-${POSTGRES_VERSION}.jar /opt/hive/lib/postgresql-${POSTGRES_VERSION}.jar
ADD https://repo.maven.apache.org/maven2/org/apache/hadoop/hadoop-aws/${HADOOP_AWS_VERSION}/hadoop-aws-${HADOOP_AWS_VERSION}.jar /opt/hive/lib/
ADD https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-bundle/${AWS_SDK_BUNDLE_VERSION}/aws-java-sdk-bundle-${AWS_SDK_BUNDLE_VERSION}.jar /opt/hive/lib/

COPY conf/hive-site.xml /opt/hive/conf/hive-site.xml
COPY conf/core-site.xml /opt/hive/conf/core-site.xml

ENV HADOOP_HOME=/opt/hadoop
ENV HIVE_HOME=/opt/hive
ENV HIVE_CONF_DIR=/opt/hive/conf
ENV HADOOP_CONF_DIR=/opt/hive/conf
ENV PATH=/opt/hadoop/bin:/opt/hive/bin:$PATH

WORKDIR /opt/hive
