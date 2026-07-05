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

# Build cometvisu-knxd-fcgi (FastCGI backend for CometVisu Protocol)
RUN git clone https://github.com/CometVisu/cometvisu-knxd-fcgi.git \
 && cd cometvisu-knxd-fcgi \
 && cmake -B build -DCMAKE_BUILD_TYPE=Release -DKNXD_SOURCE_DIR=$SOURCEDIR/knxd/src \
 && cmake --build build -j$(nproc) \
 && cmake --install build --prefix /tmp/cometvisu-knxd-fcgi-install \
 && ./build/src/cometvisu-knxd-fcgi --version | awk '{print $NF}' > /tmp/knxd-fcgi-version \
 && git rev-parse --short HEAD > /tmp/knxd-fcgi-git-rev

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
 && apt-get install -y --no-install-recommends rrdtool libxml2-dev libusb-1.0-0 libev4t64 libfmt10 libinih1 libfcgi0t64 libapache2-mod-fcgid \
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
ARG GIT_REV_KNXD_FCGI=local
ARG KNXD_FCGI_VERSION=unknown

# Own labels
LABEL maintainer="https://www.cometvisu.org/"
LABEL org.cometvisu.knxd.version="0.14.75"
LABEL org.cometvisu.knxd-fcgi.version="$KNXD_FCGI_VERSION"
LABEL org.cometvisu.cometvisubase.version="$BUILD_DATE-$VCS_REF"
LABEL org.cometvisu.cometvisubase.build-date="$BUILD_DATE_COMETVISUBASE"
LABEL org.cometvisu.cometvisubase.git-rev="$GIT_REV_COMETVISUBASE"
LABEL org.cometvisu.knxd.git-rev="$GIT_REV_KNXD"
LABEL org.cometvisu.knxd-fcgi.git-rev="$GIT_REV_KNXD_FCGI"
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
COPY --from=builder /tmp/cometvisu-knxd-fcgi-install/bin/cometvisu-knxd-fcgi /usr/local/bin/cometvisu-knxd-fcgi
COPY --from=builder /tmp/knxd-fcgi-git-rev /usr/share/cometvisu-knxd-fcgi-git-rev

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
 && a2enmod fcgid \
 && { \
    echo 'FcgidWrapper /usr/local/bin/cometvisu-knxd-fcgi virtual'; \
    echo 'FcgidCmdOptions /usr/local/bin/cometvisu-knxd-fcgi \'; \
    echo '  IOTimeout 310 \'; \
    echo '  MaxProcesses 1 \'; \
    echo '  MinProcesses 1 \'; \
    echo '  ProcessLifeTime 0 \'; \
    echo '  IdleTimeout 600'; \
    } | tee "$APACHE_CONFDIR/conf-available/cv-knxd-fcgi.conf" \
 && { \
    echo '<LocationMatch "^/cgi-bin/(l|r|w)$">'; \
    echo '    SetHandler fcgid-script'; \
    echo '    Options +ExecCGI'; \
    echo '</LocationMatch>'; \
    } | tee -a "$APACHE_CONFDIR/conf-available/cv-knxd-fcgi.conf" \
 && a2enconf cv-knxd-fcgi

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

ENV CGI_URL_PATH=/cgi-bin/
ENV BACKEND_NAME="knxd"
ENV BACKEND_KNXD=""
ENV BACKEND_MQTT=""
ENV BACKEND_OPENHAB=""
ENV BACKEND_PROXY_SOURCE=""
ENV BACKEND_PROXY_TARGET=""

ENV STOP_ON_BAD_HEALTH=true

EXPOSE 80/tcp

HEALTHCHECK --interval=120s --timeout=10s --start-period=10s CMD ["healthcheck"]
CMD ["apache2-foreground"]
