# Build the knxd Docker container
# 
# note: debian:jessie is not the smallest possible environment - but as it
# is expected that also an Apache with PHP is run the same environment was
# chosen to have an overall optimization.
# An obvious choice for a smaller enviroment could be e.g. debian:stable-slim

FROM debian:buster as builder

ARG BUILD_BRANCH
RUN echo BUILD_BRANCH: ${BUILD_BRANCH}
# debug
#ARG MY_ARG
#ARG MY_ARG2=default4MA2
#ARG SOURCE_BRANCH
#ARG SOURCE_COMMIT
#ARG COMMIT_MSG
#ARG DOCKER_REPO
#ARG CACHE_TAG
#ARG IMAGE_NAME
#RUN echo MY_ARG: ${MY_ARG}
#RUN echo MY_ARG2: ${MY_ARG2}
#RUN echo MY_ARG3: ${MY_ARG3:-def4ma3}
#RUN echo SOURCE_BRANCH: ${SOURCE_BRANCH}
#RUN echo SOURCE_COMMIT: ${SOURCE_COMMIT}
#RUN echo COMMIT_MSG: ${COMMIT_MSG}
#RUN echo DOCKER_REPO: ${DOCKER_REPO}
#RUN echo CACHE_TAG: ${CACHE_TAG}
#RUN echo IMAGE_NAME: ${IMAGE_NAME}
# debug

ENV BUILD_PACKAGES \
  build-essential \
  devscripts \
  equivs \
  debhelper \
  autoconf \
  automake \
  libtool \
  git \
  libusb-1.0-0-dev \
  pkg-config \
  libsystemd-dev \
  libev-dev \
  libfmt-dev
RUN apt-get update && apt-get -y install --no-install-recommends $BUILD_PACKAGES

# build knxd
ENV KNXDIR /usr
ENV INSTALLDIR $KNXDIR/local

RUN git clone https://github.com/knxd/knxd.git && cd knxd && git checkout ${BUILD_BRANCH} \
  && sh bootstrap.sh \
  && ./configure --enable-silent-rules --enable-eibnetip --enable-eibnetiptunnel --disable-eibnetipserver --disable-systemd --enable-groupcache --disable-java \
    --disable-shared --enable-static \
    --prefix=$INSTALLDIR/ \
  && make \
  && make install-strip \
  && cp src/server/knxd_args /usr/local/bin

# install in a fresh and thus clean environment

FROM debian:buster

LABEL maintainer="Christian Mayer docker@cometvisu.org" \
  description="The knxd to access the KNX bus."

RUN apt-get update \
  && apt-get -y install --no-install-recommends libev4 libusb-1.0 \
  && apt-get clean && rm -rf /var/lib/apt/lists/*

COPY --from=builder /usr/local/bin/ /usr/local/libexec/knxd/* /usr/local/bin/
COPY knxd-entrypoint /usr/local/bin/knxd-entrypoint
ENTRYPOINT ["knxd-entrypoint"]

ENV KNX_INTERFACE iptn:172.17.0.1:3700
ENV KNX_PA 1.1.238
ENV KNX_CLIENT_PAS 1.1.239:1
ENV KNXD_PARAMETERS ""

EXPOSE 6720

CMD knxd /etc/knxd.conf
