FROM php:8.5-apache-trixie AS builder

##################
# Compile knxd (without systemd)
RUN DEBIAN_FRONTEND=noninteractive apt-get -qq update \
 && apt-get install --no-install-recommends -y \
    git build-essential autoconf automake libtool \
    pkg-config libev-dev libfmt-dev libinih-dev libusb-1.0-0-dev \
    cmake libfcgi-dev

ENV SOURCEDIR=/usr/src

WORKDIR $SOURCEDIR

RUN git clone -b debian https://github.com/knxd/knxd.git \
 && cd knxd \
 && sh ./bootstrap.sh \
 && ./configure --disable-systemd --prefix=/usr --sysconfdir=/etc --libexecdir=/usr/lib \
 && make -j$(nproc) \
 && make install DESTDIR=/tmp/knxd-install \
 && git rev-parse --short HEAD > /tmp/knxd-git-rev

# Build cometvisu-knx-fcgi 2.x (FastCGI backend for the CometVisu Protocol)
RUN git clone https://github.com/CometVisu/cometvisu-knx-fcgi.git \
 && cd cometvisu-knx-fcgi \
 && cmake -B build -DCMAKE_BUILD_TYPE=Release -DKNXD_SOURCE_DIR=$SOURCEDIR/knxd/src \
 && cmake --build build -j$(nproc) \
 && cmake --install build --prefix /tmp/cometvisu-knx-fcgi-install \
 && ./build/src/cometvisu-knx-fcgi --version | awk '{print $NF}' > /tmp/knx-fcgi-version \
 && git rev-parse --short HEAD > /tmp/knx-fcgi-git-rev

##############
# Run environment
FROM php:8.5-apache-trixie

# clear upstream spilled ENV
ENV PHPIZE_DEPS=
ENV PHP_CFLAGS=
ENV PHP_CPPFLAGS=
ENV PHP_LDFLAGS=
ENV GPG_KEYS=
ENV PHP_URL=
ENV PHP_ASC_URL=
ENV PHP_SHA256=

# install the non changing parts
RUN DEBIAN_FRONTEND=noninteractive apt-get -qq update \
 && apt-get install -y --no-install-recommends rrdtool libxml2-dev libusb-1.0-0 libev4t64 libfmt10 libinih1 libfcgi0t64 \
 && docker-php-ext-install soap \
 && docker-php-ext-install pdo_mysql \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /usr/share/doc/*

# set by the automated build at DockerHub
ARG BUILD_DATE
ARG VCS_REF

# Build-time arguments provided by CI or local build script
ARG BUILD_DATE_COMETVISUBASE
ARG GIT_REV_COMETVISUBASE
ARG GIT_REV_KNXD
ARG GIT_REV_KNX_FCGI=local
ARG KNX_FCGI_VERSION=unknown

# Own labels
LABEL maintainer="https://www.cometvisu.org/"
LABEL org.cometvisu.knxd.version="0.14.75"
LABEL org.cometvisu.knx-fcgi.version="$KNX_FCGI_VERSION"
LABEL org.cometvisu.cometvisubase.version="$BUILD_DATE-$VCS_REF"
LABEL org.cometvisu.cometvisubase.build-date="$BUILD_DATE_COMETVISUBASE"
LABEL org.cometvisu.cometvisubase.git-rev="$GIT_REV_COMETVISUBASE"
LABEL org.cometvisu.knxd.git-rev="$GIT_REV_KNXD"
LABEL org.cometvisu.knx-fcgi.git-rev="$GIT_REV_KNX_FCGI"
# Labels according to http://label-schema.org/rc1/
LABEL org.label-schema.schema-version="1.0"
LABEL org.label-schema.build-date=$BUILD_DATE
LABEL org.label-schema.name="CometVisu"
LABEL org.label-schema.description="The CometVisu visualization abstract base"
LABEL org.label-schema.usage="README.md"
LABEL org.label-schema.url="https://www.cometvisu.org"
LABEL org.label-schema.vcs-url="https://github.com/CometVisu/Docker"
LABEL org.label-schema.vcs-ref=$VCS_REF
LABEL org.label-schema.vendor="The CometVisu project"
LABEL org.label-schema.version="$BUILD_DATE-$VCS_REF"

COPY --from=builder /tmp/knxd-install/usr/ /usr/
COPY --from=builder /usr/src/knxd/systemd/knxd.conf /etc/knxd.conf
COPY --from=builder /tmp/knxd-git-rev /usr/share/knxd/git-rev
COPY --from=builder /tmp/cometvisu-knx-fcgi-install/bin/cometvisu-knx-fcgi /usr/local/bin/cometvisu-knx-fcgi
COPY --from=builder /tmp/knx-fcgi-git-rev /usr/share/cometvisu-knx-fcgi-git-rev

COPY rrdfetch /usr/lib/cgi-bin/rrdfetch

##################

# Overwrite package default - allow index
RUN { \
    echo 'LimitRequestFieldSize 32768'; \
    echo 'LimitRequestLine 32768'; \
    echo; \
    echo '<FilesMatch \.php$>'; \
    echo '\tSetHandler application/x-httpd-php'; \
    echo '</FilesMatch>'; \
    echo; \
    echo 'DirectoryIndex enabled'; \
    echo 'DirectoryIndex index.php index.html'; \
    echo; \
    echo '<Directory /var/www/>'; \
    echo '\tOptions +Indexes'; \
    echo '\tAllowOverride All'; \
    echo '</Directory>'; \
    } | tee "$APACHE_CONFDIR/conf-available/cv-docker-php.conf" \
 && a2disconf docker-php \
 && a2enconf cv-docker-php \
 && mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini" \
 && { \
    echo 'upload_max_filesize = 40M'; \
    echo 'post_max_size = 40M'; \
    } | tee "$PHP_INI_DIR/conf.d/cv-php.ini" \
 && chmod +x /usr/lib/cgi-bin/rrdfetch \
 && a2enmod cgi \
 && a2enmod headers \
 && a2enmod proxy \
 && a2enmod proxy_fcgi \
 && a2enconf serve-cgi-bin \
 && { \
    echo '# Proxy FastCGI requests to the standalone cometvisu-knx-fcgi Unix socket'; \
    echo 'ProxyPass "/cgi-bin/l" "unix:/run/cometvisu-fcgi.sock|fcgi://localhost/l"'; \
    echo 'ProxyPass "/cgi-bin/r" "unix:/run/cometvisu-fcgi.sock|fcgi://localhost/r"'; \
    echo 'ProxyPass "/cgi-bin/w" "unix:/run/cometvisu-fcgi.sock|fcgi://localhost/w"'; \
    } | tee "$APACHE_CONFDIR/conf-available/cv-knx-fcgi.conf" \
 && a2enconf cv-knx-fcgi

COPY cometvisu-entrypoint /usr/local/bin/cometvisu-entrypoint
ENTRYPOINT ["cometvisu-entrypoint"]

ENV ACCESS_LOG=false
# add knxtools to PATH (e.g. groupwrite, groupread)
ENV PATH="$PATH:/usr/lib/knxd"

ENV KNX_INTERFACE=""
ENV KNX_INTERFACE_OPTIONS="-B single -A retry-delay=1"
ENV KNX_PA=1.1.238
ENV KNX_CLIENT_PAS=1.1.239:10
ENV KNXD_PARAMETERS=""

ENV FCGI_SOCKET=/run/cometvisu-fcgi.sock
ENV CGI_URL_PATH=/cgi-bin/
# knxd socket path - only used when NO_ENTRYPOINT_KNXD=extern
ENV KNXD_SOCKET=/run/knx
ENV BACKEND_NAME="knxd"
ENV BACKEND_KNXD=""
ENV BACKEND_MQTT=""
ENV BACKEND_OPENHAB=""
ENV BACKEND_PROXY_SOURCE=""
ENV BACKEND_PROXY_TARGET=""

# settings for the FastCGI backend (cometvisu-knx-fcgi 2.x)
# The backend requires an explicit transport; this container always bundles knxd.
ENV KNX_TRANSPORT=knxd
ENV LONGPOLL_TIMEOUT_SEC=300
# Worker processes.  Since 2.x the workers never open a knxd connection of
# their own - the single cache reader uses one, whatever FCGI_WORKERS is.
# At least 2 workers are required (the backend keeps two of them free for
# /w and /l), so with 2 workers exactly one long-poll can be served.
ENV FCGI_WORKERS=9
# Outgoing telegram rate limit over all clients; protects the KNX bus.
ENV KNX_TX_MAX_PER_SEC=10
ENV DEBUG_BACKEND=

ENV STOP_ON_BAD_HEALTH=true

# Port Apache listens on inside the container. Changing it is an expert option -
# EXPOSE stays 80, so use "-p <host-port>:${APACHE_PORT}" when publishing.
ENV APACHE_PORT=80

EXPOSE 80/tcp

HEALTHCHECK --interval=120s --timeout=10s --start-period=10s CMD ["healthcheck"]
CMD ["apache2-foreground"]
