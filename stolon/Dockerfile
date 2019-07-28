
ARG PG_VERSION
############################
# Build tools binaries in separate image
############################
ARG GO_VERSION=1.12.7
FROM golang:${GO_VERSION}-alpine AS tools

ENV TOOLS_VERSION 0.7.0

RUN apk update && apk add --no-cache git \
    && mkdir -p ${GOPATH}/src/github.com/timescale/ \
    && cd ${GOPATH}/src/github.com/timescale/ \
    && git clone https://github.com/timescale/timescaledb-tune.git \
    && git clone https://github.com/timescale/timescaledb-parallel-copy.git \
    # Build timescaledb-tune
    && cd timescaledb-tune/cmd/timescaledb-tune \
    && git fetch && git checkout --quiet $(git describe --abbrev=0) \
    && go get -d -v \
    && go build -o /go/bin/timescaledb-tune \
    # Build timescaledb-parallel-copy
    && cd ${GOPATH}/src/github.com/timescale/timescaledb-parallel-copy/cmd/timescaledb-parallel-copy \
    && git fetch && git checkout --quiet $(git describe --abbrev=0) \
    && go get -d -v \
    && go build -o /go/bin/timescaledb-parallel-copy

############################
# Build old versions in a separate stage
############################
ARG PG_VERSION
FROM sorintlab/stolon:master-pg10 AS oldversions
ARG PG_VERSION

USER 0
RUN set -ex \
    && mkdir -p /var/lib/apt/lists/partial \
    && apt-get update \
    && apt-get -y install \
            build-essential \
            apt-utils \
            git \
            gcc \
            make \
            cmake \
            wget \
            dpkg-dev \
            libc-dev \
            libssl-dev \
            libpq-dev \
            postgresql-server-dev-10 \
    && mkdir -p /build/ \
    && git clone https://github.com/timescale/timescaledb /build/timescaledb \
    \
    && cd /build/timescaledb \
    # This script is a bit ugly, but once all the old versions are buildable
    # on PG10, we can remove the 'if' guard
    && echo "if [ \"$(echo ${PG_VERSION} | cut -c1-2)\" != \"10\" ] || [ "\${OLD_VERSION}" \> "1.0.1" ]; then cd /build/timescaledb && rm -fr build && git reset HEAD --hard && git fetch && git checkout \${OLD_VERSION} && ./bootstrap -DPROJECT_INSTALL_METHOD=\"docker\" && cd bu$
    && chmod +x ./build_old.sh

#####
# Add the latest previous version to the end of the list for each new build
#####
RUN OLD_VERSION=1.1.0 /build/timescaledb/build_old.sh
RUN OLD_VERSION=1.1.1 /build/timescaledb/build_old.sh
RUN OLD_VERSION=1.2.0 /build/timescaledb/build_old.sh
RUN OLD_VERSION=1.2.1 /build/timescaledb/build_old.sh
RUN OLD_VERSION=1.2.2 /build/timescaledb/build_old.sh
RUN OLD_VERSION=1.3.0 /build/timescaledb/build_old.sh
RUN OLD_VERSION=1.3.1 /build/timescaledb/build_old.sh
RUN OLD_VERSION=1.3.2 /build/timescaledb/build_old.sh

# Cleanup
RUN \
    # Remove update files and mock files; not needed for old versions
    echo $(pg_config --pkglibdir) \
    && rm -f $(pg_config --sharedir)/extension/timescaledb--*--*.sql \
    && rm -f $(pg_config --sharedir)/extension/timescaledb*mock*.sql \
    # Remove all but the last several versiosn ()
    && KEEP_NUM_VERSIONS=5 \
    && rm -f $(ls -1 $(pg_config --pkglibdir)/timescaledb-*.so | head -n -${KEEP_NUM_VERSIONS}) \
    && rm -f $(ls -1 $(pg_config --sharedir)/extension/timescaledb-*.sql | head -n -${KEEP_NUM_VERSIONS}) \
    # Clean up the rest of the image
    && cd ~ \
    && apt-get autoremove --purge -y \
            build-essential \
            libssl-dev \
            dpkg-dev \
            gcc \
            libc-dev \
            make \
            cmake \
    && apt-get clean -y \
    && rm -rf \
      "${HOME}/.cache" \
        /var/lib/apt/lists/* \
        /tmp/*               \
        /var/tmp/*

############################
# Now build image and copy in tools
############################
ARG PG_VERSION
FROM sorintlab/stolon:master-pg10
ARG PG_VERSION

MAINTAINER Timescale https://www.timescale.com

ENV TIMESCALEDB_VERSION 1.4.0

COPY docker-entrypoint-initdb.d/* /docker-entrypoint-initdb.d/
COPY --from=tools /go/bin/* /usr/local/bin/
COPY --from=oldversions /usr/lib/postgresql/10/lib/timescaledb-*.so /usr/local/lib/postgresql/
COPY --from=oldversions /usr/share/postgresql/10/extension/timescaledb--*.sql /usr/local/share/postgresql/extension/

USER 0
RUN set -ex \
    && mkdir -p /var/lib/apt/lists/partial \
    && apt-get update \
    && apt-get -y install \
            build-essential \
            apt-utils \
            git \
            gcc \
            make \
            cmake \
            wget \
            dpkg-dev \
            libc-dev \
            libssl-dev \
            libpq-dev \
            postgresql-server-dev-10 \
    && mkdir -p /build/ \
    && git clone https://github.com/timescale/timescaledb /build/timescaledb \
    \
    # Build current version \
    && cd /build/timescaledb && rm -fr build \
    && git checkout ${TIMESCALEDB_VERSION} \
    && ./bootstrap -DPROJECT_INSTALL_METHOD="docker" \
    && cd build && make install \
    && cd ~ \
    && apt-get autoremove --purge -y \
            build-essential \
            apt-utils \
            git \
            gcc \
            make \
            cmake \
            wget \
            dpkg-dev \
            libc-dev \
            libssl-dev \
            libpq-dev \
            postgresql-server-dev-10 \
    && apt-get clean -y \
    && rm -rf \
      "${HOME}/.cache" \
        /var/lib/apt/lists/* \
        /tmp/*               \
        /var/tmp/*
RUN sed -r -i "s/[#]*\s*(shared_preload_libraries)\s*=\s*'(.*)'/\1 = 'timescaledb,\2'/;s/,'/'/" /usr/share/postgresql/10/postgresql.conf.sample