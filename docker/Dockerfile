FROM alpine:3.12 AS builder

# Check required arguments exist. These will be provided by the Github Action
# Workflow and are required to ensure the correct branches are being used.
ARG SHAIRPORT_SYNC_BRANCH
RUN test -n "$SHAIRPORT_SYNC_BRANCH"
ARG NQPTP_BRANCH
RUN test -n "$NQPTP_BRANCH"

RUN apk -U add \
        git \
        build-base \
        autoconf \
        automake \
        libtool \
        dbus \
        alsa-lib-dev \
        popt-dev \
        soxr-dev \
        avahi-dev \
        libconfig-dev \
        libsndfile-dev \
        mosquitto-dev \
        libsodium-dev \
        libgcrypt-dev \
        ffmpeg-dev \
        xxd \
        libressl-dev \
        openssl-dev \
        libplist-dev

##### ALAC #####
RUN git clone https://github.com/mikebrady/alac
WORKDIR /alac
RUN autoreconf -i
RUN ./configure
RUN make
RUN make install
WORKDIR /
##### ALAC END #####

##### NQPTP #####
RUN git clone https://github.com/mikebrady/nqptp
WORKDIR /nqptp
RUN git checkout "$NQPTP_BRANCH"
RUN autoreconf -i
RUN ./configure
RUN make
RUN make install
WORKDIR /
##### NQPTP END #####

##### SPS #####
WORKDIR /shairport-sync
COPY . .
RUN git checkout "$SHAIRPORT_SYNC_BRANCH"
WORKDIR /shairport-sync/build
RUN autoreconf -i ../
RUN ../configure --sysconfdir=/etc --with-alsa --with-soxr --with-avahi --with-ssl=openssl --with-airplay-2 \
        --with-metadata --with-dummy --with-pipe --with-dbus-interface \
        --with-stdout --with-mpris-interface --with-mqtt-client \
        --with-apple-alac --with-convolution
RUN make -j $(nproc)
RUN DESTDIR=install make install
WORKDIR /
##### SPS END #####

# Shairport Sync Runtime System
FROM crazymax/alpine-s6:3.12-3.1.1.2

RUN apk -U add \
        alsa-lib \
        dbus \
        popt \
        glib \
        soxr \
        avahi \
        avahi-tools \
        libconfig \
        libsndfile \
        mosquitto \
        libuuid \
        ffmpeg \
        libsodium \
        libgcrypt \
        libplist \
        mandoc \
        man-pages \
        less \
        less-doc

# Copy build files.
COPY --from=builder /shairport-sync/build/install/usr/local/bin/shairport-sync /usr/local/bin/shairport-sync
COPY --from=builder /shairport-sync/build/install/usr/local/share/man/man7 /usr/share/man/man7
COPY --from=builder /usr/local/bin/nqptp /usr/local/bin/nqptp
COPY --from=builder /usr/local/lib/libalac.* /usr/local/lib/
COPY --from=builder /shairport-sync/build/install/etc/shairport-sync.conf /etc/
COPY --from=builder /shairport-sync/build/install/etc/shairport-sync.conf.sample /etc/
COPY --from=builder /shairport-sync/build/install/etc/dbus-1/system.d/shairport-sync-dbus.conf /etc/dbus-1/system.d/
COPY --from=builder /shairport-sync/build/install/etc/dbus-1/system.d/shairport-sync-mpris.conf /etc/dbus-1/system.d/

COPY ./docker/etc/s6-overlay/s6-rc.d /etc/s6-overlay/s6-rc.d
RUN chmod +x /etc/s6-overlay/s6-rc.d/startup/script.sh

# Create non-root user for running the container -- running as the user 'shairport-sync' also allows
# Shairport Sync to provide the D-Bus and MPRIS interfaces within the container

RUN addgroup shairport-sync
RUN adduser -D shairport-sync -G shairport-sync

# Add the shairport-sync user to the pre-existing audio group, which has ID 29, for access to the ALSA stuff
RUN addgroup -g 29 docker_audio && addgroup shairport-sync docker_audio && addgroup shairport-sync audio

# Remove anything we don't need.
RUN rm -rf /lib/apk/db/*

ENTRYPOINT [ "/init", "/usr/local/bin/shairport-sync" ]
