# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.
ARG OWNER=jupyter
ARG BASE_CONTAINER=$OWNER/scipy-notebook
FROM $BASE_CONTAINER

LABEL maintainer="Jupyter Project <jupyter@googlegroups.com>"

# Fix: https://github.com/hadolint/hadolint/wiki/DL4006
# Fix: https://github.com/koalaman/shellcheck/wiki/SC3014
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER root

# Spark dependencies
# Default values can be overridden at build time
# (ARGS are in lower case to distinguish them from ENV)
ARG spark_version="3.2.2"
ARG hadoop_version="3.2"
ARG scala_version
ARG spark_checksum="a7bdcaf598e9bcf78d7cbd2b8ea08d4363c45a4b0cda0940e168ef7d592459df1dde0c33143049d58b61af15c83e2ea2a93bcf6ec63df46b693a36c978d57182"
ARG openjdk_version="8"
ARG rapids_ver=22.10.0
ARG hadoop_aws_ver=3.3.1
ARG aws_java_sdk_bundle_ver=1.11.655
ARG gcs_connector_ver=2.2.2

ENV APACHE_SPARK_VERSION="${spark_version}" \
    HADOOP_VERSION="${hadoop_version}"

RUN apt-get update -y
RUN apt-get update --yes && \
    apt-get install --yes --no-install-recommends \
    "openjdk-${openjdk_version}-jre-headless" \
    ca-certificates-java && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# RUN apt-get update && apt-get install -y --no-install-recommends openjdk-8-jdk openjdk-8-jre gpg gpg-agent wget
# ENV JAVA_HOME /usr/lib/jvm/java-1.8.0-openjdk-amd64
# ENV PATH $PATH:/usr/lib/jvm/java-1.8.0-openjdk-amd64/jre/bin:/usr/lib/jvm/java-1.8.0-openjdk-amd64/bin

# Spark installation
WORKDIR /tmp

RUN if [ -z "${scala_version}" ]; then \
    wget -qO "spark.tgz" "https://archive.apache.org/dist/spark/spark-${APACHE_SPARK_VERSION}/spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz"; \
  else \
    wget -qO "spark.tgz" "https://archive.apache.org/dist/spark/spark-${APACHE_SPARK_VERSION}/spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}-scala${scala_version}.tgz"; \
  fi && \
  echo "${spark_checksum} *spark.tgz" | sha512sum -c - && \
  tar xzf "spark.tgz" -C /usr/local --owner root --group root --no-same-owner && \
  rm "spark.tgz"

# Configure Spark
ENV SPARK_HOME=/usr/local/spark
ENV SPARK_OPTS="--driver-java-options=-Xms1024M --driver-java-options=-Xmx4096M --driver-java-options=-Dlog4j.logLevel=info" \
    PATH="${PATH}:${SPARK_HOME}/bin"

RUN if [ -z "${scala_version}" ]; then \
    ln -s "spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}" "${SPARK_HOME}"; \
  else \
    ln -s "spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}-scala${scala_version}" "${SPARK_HOME}"; \
  fi && \
  # Add a link in the before_notebook hook in order to source automatically PYTHONPATH && \
  mkdir -p /usr/local/bin/before-notebook.d && \
  ln -s "${SPARK_HOME}/sbin/spark-config.sh" /usr/local/bin/before-notebook.d/spark-config.sh

# Configure IPython system-wide
COPY ipython_kernel_config.py "/etc/ipython/"
RUN fix-permissions "/etc/ipython/"

WORKDIR /usr/local/spark/jars
# Install rapids plugin jar
RUN set -ex && \
    wget https://repo1.maven.org/maven2/com/nvidia/rapids-4-spark_2.12/$rapids_ver/rapids-4-spark_2.12-$rapids_ver.jar && \
    echo "a604ee7daca1930e06a3f39afae04e943985f702 rapids-4-spark_2.12-22.10.0.jar" | sha1sum -c -

# Add necessary jars to support AWS s3 storage access
RUN set -ex && \
    wget https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-aws/$hadoop_aws_ver/hadoop-aws-$hadoop_aws_ver.jar && \
    wget https://repo1.maven.org/maven2/com/amazonaws/aws-java-sdk-bundle/$aws_java_sdk_bundle_ver/aws-java-sdk-bundle-$aws_java_sdk_bundle_ver.jar
# Add necessary jars to support GCS storage access
RUN set -ex && \
    wget https://repo1.maven.org/maven2/com/google/cloud/bigdataoss/gcs-connector/hadoop3-$gcs_connector_ver/gcs-connector-hadoop3-$gcs_connector_ver-shaded.jar
# TODO: Add necessary jars to support Azure storage access

USER ${NB_UID}

# Install pyarrow
RUN mamba install --quiet --yes \
    'pyarrow' && \
    mamba clean --all -f -y && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

WORKDIR "${HOME}"
# Every new spark context that is created is put onto an incrementing port (ie. 4040, 4041, 4042, etc.),
# and it might be necessary to open multiple ports.
EXPOSE 4040 4041 4042 4043