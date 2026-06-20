FROM almalinux:9 AS builder

ENV CARGO_NET_GIT_FETCH_WITH_CLI=true

RUN \
        dnf -y update && \
        dnf -y install epel-release dnf-plugins-core && \
        dnf config-manager --set-enabled crb

RUN dnf -y install \
        autoconf \
        automake \
        cargo \
        cbindgen \
        diffutils \
        patch \
        dpdk-devel \
        elfutils-libelf-devel \
        file \
        file-devel \
        gcc \
        gcc-c++ \
        git \
        hiredis-devel \
        jansson-devel \
        jq \
        libbpf-devel \
        libtool \
        libyaml-devel \
        libnfnetlink-devel \
        libnetfilter_queue-devel \
        libnet-devel \
        libcap-ng-devel \
        libevent-devel \
        libmaxminddb-devel \
        libpcap-devel \
        libprelude-devel \
        libtool \
        lz4-devel \
        make \
        numactl-devel \
        pcre2-devel \
        pkgconfig \
        python3-devel \
        python3-yaml \
        rust \
        which \
        zlib-devel
# nDPI
RUN dnf -y install \
        wget \
        unzip \
#        curl \
#        build-essential \
        gcc gcc-c++ make \
        gettext \
        flex \
        bison \
        libpcap-devel \
        json-c-devel \
        numactl-devel \
        pcre2-devel \
        libmaxminddb-devel \
        rrdtool-devel

RUN mkdir -p /usr/local/src
WORKDIR /usr/local/src
RUN wget https://github.com/ntop/nDPI/archive/refs/tags/4.14.zip
RUN unzip -tq 4.14.zip && unzip -q 4.14.zip
WORKDIR /usr/local/src/nDPI-4.14
# Building
RUN ./autogen.sh
RUN ./configure --prefix=/usr/local --with-pic CFLAGS="-march=native -flto=auto -ffat-lto-objects -std=c11"
RUN make -j$(nproc)
RUN make install
RUN ldconfig
# End nDPI

RUN if [ "$(arch)" = "x86_64" ]; then \
    dnf -y install hyperscan-devel; \
fi

ARG VERSION

WORKDIR /src

RUN if [ "${VERSION}" = "master" ]; then \
        git clone https://github.com/OISF/suricata.git suricata-${VERSION}; \
        (cd suricata-${VERSION}/suricata-update && \
                curl -L https://github.com/OISF/suricata-update/archive/master.tar.gz | tar zxf - --strip-components=1); \
        (cd suricata-${VERSION} && ./autogen.sh); \
else \
        curl -OL https://www.openinfosecfoundation.org/download/suricata-${VERSION}.tar.gz; \
        tar zxf suricata-${VERSION}.tar.gz; \
fi

WORKDIR /src/suricata-${VERSION}

COPY /profiling.patch /tmp/profiling.patch
RUN patch -p1 < /tmp/profiling.patch

ARG CONFIGURE_ARGS

RUN ./configure \
        --prefix=/usr \
#        --disable-shared \
        --disable-gccmarch-native \
        --enable-nfqueue \
        --enable-hiredis \
        --enable-geoip \
        --enable-ebpf \
        --enable-dpdk \
        --enable-profiling-rules \
        # nDPI
        --enable-ndpi \
        --with-ndpi=/usr/local/src/nDPI-4.14 \
        ${CONFIGURE_ARGS}

ARG CORES=2

RUN make -j "${CORES}"

# Verify Hyperscan support is enabled.
RUN if [ "$(arch)" = "x86_64" ]; then \
    ./src/suricata --build-info | grep Hyperscan | grep yes; \
fi

RUN make install install-conf DESTDIR=/fakeroot

# Something about the Docker mounts won't let us copy /var/run in the
# next stage.
RUN rm -rf /fakeroot/var

FROM almalinux/9-base:latest AS runner

# Label docker image
ARG VERSION
LABEL maintainer="DevDotNet.Org <anton@devdotnet.org>"
LABEL build_version="Image version: Suricata ${VERSION}, nDPI 4.14"
LABEL org.opencontainers.image.source="https://github.com/devdotnetorg/docker-suricata"

RUN \
        dnf -y update && \
        dnf -y install epel-release && \
        dnf -y install \
        cronie \
        dpdk \
        elfutils-libelf \
        file \
        findutils \
        hiredis \
        iproute \
        jansson \
        libbpf \
        libyaml \
        libnfnetlink \
        libnetfilter_queue \
        libnet \
        libcap-ng \
        libevent \
        libmaxminddb \
        libpcap \
        libprelude \
        logrotate \
        lz4 \
        net-tools \
        numactl \
        pcre2 \
        procps-ng \
        python3 \
        python3-yaml \
        tcpdump \
        which \
        zlib && \
        if [ "$(arch)" = "x86_64" ]; then dnf -y install hyperscan; fi && \
        dnf clean all && \
        find /etc/logrotate.d -type f -not -name suricata -delete

COPY --from=builder /fakeroot /

# Create the directories that didn't get copied from the previous stage.
RUN mkdir -p /var/log/suricata /var/run/suricata /var/lib/suricata

COPY /update.yaml /etc/suricata/update.yaml
COPY /suricata.logrotate /etc/logrotate.d/suricata

RUN suricata-update update-sources

RUN useradd --system --create-home suricata && \
        chown -R suricata:suricata /etc/suricata && \
        chown -R suricata:suricata /var/log/suricata && \
        chown -R suricata:suricata /var/lib/suricata && \
        chown -R suricata:suricata /var/run/suricata && \
        cp -a /etc/suricata /etc/suricata.dist && \
        chmod 600 /etc/logrotate.d/suricata && \
        # symlink
        mkdir -p /usr/lib/suricata && \
        ln -s /usr/lib64/suricata/ndpi.so /usr/lib/suricata/ndpi.so

VOLUME /var/log/suricata
VOLUME /var/lib/suricata
VOLUME /var/run/suricata
VOLUME /etc/suricata

COPY /docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]

RUN /usr/bin/suricata --build-info
