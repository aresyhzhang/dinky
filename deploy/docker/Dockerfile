ARG FLINK_VERSION=1.19
ARG DINKY_VERSION=1.1.1-SNAPSHOT

FROM swr.cn-east-3.myhuaweicloud.com/kubesre/docker.io/flink:1.19.0-scala_2.12-java8-linux-amd64 as flink-base

FROM  swr.cn-east-3.myhuaweicloud.com/kubesre/docker.io/node:18.15.0-alpine3.17-linux-amd64 AS ui-build
WORKDIR /build/

ENV NODE_OPTIONS=--openssl-legacy-provider
ENV UMI_ENV=production

# 单独分离 package.json，是为了安装依赖可最大限度利用缓存
ADD ./dinky-web/package.json /build/package.json
RUN npm config set registry https://registry.npmmirror.com
RUN npm install --legacy-peer-deps
ADD ./dinky-web .
RUN npm run build

FROM swr.cn-east-3.myhuaweicloud.com/kubesre/docker.io/maven:3.9-eclipse-temurin-8-alpine-linux-amd64 AS build
WORKDIR /build/
ARG FLINK_VERSION=1.19
ARG DINKY_VERSION=1.1.1-SNAPSHOT
ENV FLINK_VERSION=${FLINK_VERSION}
ENV DINKY_VERSION=${DINKY_VERSION}

ADD . .
COPY --from=ui-build /build/dist/ /build/dinky-web/dist/

RUN mvn package -Dmaven.test.skip=true -P prod,flink-single-version,flink-${FLINK_VERSION},fast
RUN mkdir release &&  \
    tar -C release -xvf build/dinky-release-${FLINK_VERSION}-${DINKY_VERSION}.tar.gz && \
    mv release/dinky-release-* release/dinky


FROM swr.cn-east-3.myhuaweicloud.com/kubesre/docker.io/eclipse-temurin:8-jre-jammy-linux-amd64

ARG FLINK_VERSION=1.19
ENV FLINK_VERSION=${FLINK_VERSION}
ENV DINKY_HOME=/opt/dinky/
ENV H2_DB=./tmp/db/h2

WORKDIR /opt/dinky/

USER root

COPY --from=build /build/release/dinky /opt/dinky/
COPY --from=flink-base  /opt/flink/lib/*.jar  /opt/dinky/extends/flink${FLINK_VERSION}/flink/
RUN rm -f  /opt/dinky/extends/flink${FLINK_VERSION}/flink/flink-table-planner-loader*.jar

COPY --from=flink-base  /opt/flink/opt/flink-table-planner*.jar /opt/dinky/extends/flink${FLINK_VERSION}/flink/

RUN mkdir /opt/dinky/customJar && chmod -R 777 /opt/dinky/

EXPOSE 8888

CMD  ./auto.sh startOnPending