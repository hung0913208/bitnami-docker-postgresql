FROM postgres:12-bullseye AS build

ARG USE_POSTGIS=false
ENV PLUGIN_VERSION=v2.1.0.Alpha1
ENV PROTOC_VERSION=1.4
ENV PG_MAJOR=12

ENV WAL2JSON_COMMIT_ID=92b33c7d7c2fccbeb9f79455dafbc92e87e00ddd

# Install the packages which will be required to get everything to compile
RUN apt-get update \
    && apt-get install -f -y --no-install-recommends \
        software-properties-common \
        build-essential \
        pkg-config \
        git \
        postgresql-server-dev-$PG_MAJOR \
    && add-apt-repository "deb http://ftp.debian.org/debian testing main contrib" \
    && apt-get update && apt-get install -f -y --no-install-recommends \
        libprotobuf-c-dev=$PROTOC_VERSION.* \
    && rm -rf /var/lib/apt/lists/*

# Compile the plugin from sources and install it
RUN git clone https://github.com/debezium/postgres-decoderbufs -b $PLUGIN_VERSION --single-branch \
    && cd /postgres-decoderbufs \
    && make && make install \
    && cd / \
    && rm -rf postgres-decoderbufs

RUN git clone https://github.com/eulerto/wal2json -b master --single-branch \
    && cd /wal2json \
    && git checkout $WAL2JSON_COMMIT_ID \
    && make && make install \
    && cd / \
    && rm -rf wal2json

# ------------------------------------------------------------------------------ #
FROM docker.io/bitnami/minideb:bullseye
LABEL maintainer="Hung Nguyen Xuan Pham"

ARG USE_POSTGIS=false
ENV PG_MAJOR=12
ENV POSTGIS_VERSION=
ENV HOME="/" \
    OS_ARCH="amd64" \
    OS_FLAVOUR="debian-11" \
    OS_NAME="linux"

ARG EXTRA_LOCALES=""
ARG WITH_ALL_LOCALES="no"

COPY prebuildfs /

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Install required system packages and dependencies
RUN install_packages acl gpg ca-certificates curl gzip libbsd0                                                                              \
                    libbz2-1.0 libc6 libedit2 libffi7 libgcc-s1                                                                             \
                    libgmp10 libgnutls30 libhogweed6 libicu67                                                                               \
                    libidn2-0 libldap-2.4-2 liblz4-1 liblzma5                                                                               \
                    libmd0 libncurses6 libnettle8 libp11-kit0                                                                               \
                    libpcre3 libreadline8 libsasl2-2                                                                                        \
                    libsqlite3-0 libssl1.1 libstdc++6                                                                                       \
                    libtasn1-6 libtinfo6 libunistring2 libuuid1                                                                             \
                    libxml2 libxslt1.1 libzstd1 locales procps                                                                              \
                    tar zlib1g libprotobuf-c1                                                                                               \
                    software-properties-common                                                                                              \
    && add-apt-repository "deb http://ftp.debian.org/debian testing main contrib"                                                           \
    && if [ "$USE_POSTGIS" != "false" ]; then install_packages                                                                              \
        curl https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor | tee /etc/apt/trusted.gpg.d/apt.postgresql.org.gpg;        \
        echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list                 \
        postgresql-$PG_MAJOR-postgis-$POSTGIS_VERSION                                                                                       \
        postgresql-$PG_MAJOR-postgis-$POSTGIS_VERSION-scripts                                                                               \
        postgis;                                                                                                                            \
    fi
RUN . /opt/bitnami/scripts/libcomponent.sh && component_unpack "gosu" "1.14.0-152" --checksum 0c751c7e2ec0bc900a19dbec0306d6294fe744ddfb0fa64197ba1a36040092f0
RUN . /opt/bitnami/scripts/libcomponent.sh && component_unpack "postgresql" "12.11.0-158" --checksum 39b200080de55c56a68d91051668e37f9b2ac04f6dbbce91fad1bd05a89a5028
RUN apt-get update && apt-get upgrade -y && \
    rm -r /var/lib/apt/lists /var/cache/apt/archives
RUN chmod g+rwX /opt/bitnami
RUN localedef -c -f UTF-8 -i en_US en_US.UTF-8
RUN update-locale LANG=C.UTF-8 LC_MESSAGES=POSIX && \
    DEBIAN_FRONTEND=noninteractive dpkg-reconfigure locales
RUN echo 'en_GB.UTF-8 UTF-8' >> /etc/locale.gen && locale-gen
RUN echo 'en_US.UTF-8 UTF-8' >> /etc/locale.gen && locale-gen

# @NOTE: copy wal2json to bitnami image
COPY --from=build /usr/lib/postgresql/$PG_MAJOR/lib/decoderbufs.so /usr/lib/postgresql/$PG_MAJOR/lib/wal2json.so /opt/bitnami/postgresql/lib
COPY --from=build /usr/share/postgresql/$PG_MAJOR/extension/decoderbufs.control /opt/bitnami/postgresql/share/extension/

# @NOTE: copy bitnami scripts
COPY rootfs /
RUN /opt/bitnami/scripts/postgresql/postunpack.sh
RUN /opt/bitnami/scripts/locales/add-extra-locales.sh
ENV APP_VERSION="12.11.0"                                           \
    BITNAMI_APP_NAME="postgresql"                                   \
    LANG="en_US.UTF-8"                                              \
    LANGUAGE="en_US:en"                                             \
    NSS_WRAPPER_LIB="/opt/bitnami/common/lib/libnss_wrapper.so"     \
    PATH="/opt/bitnami/common/bin:/opt/bitnami/postgresql/bin:$PATH"

VOLUME [ "/bitnami/postgresql", "/docker-entrypoint-initdb.d", "/docker-entrypoint-preinitdb.d" ]

EXPOSE 5432

USER 1001
ENTRYPOINT [ "/opt/bitnami/scripts/postgresql/entrypoint.sh" ]
CMD [ "/opt/bitnami/scripts/postgresql/run.sh" ]
