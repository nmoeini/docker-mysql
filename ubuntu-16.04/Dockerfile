FROM ubuntu:xenial-20200807

LABEL name="nmoeini/mysql:5.7.31-xenial-20200807" mainteiner="Navid Moeini <nmoeiny@yahoo.com>" version="0.0.2"

ARG DEBIAN_FRONTEND
ARG GOSU_VERSION
ARG MYSQL_USER
ARG MYSQL_UID

ENV DEBIAN_FRONTEND=${DEBIAN_FRONTEND:-noninteractive} \
    GOSU_VERSION=${GOSU_VERSION:-1.12} \
    MYSQL_USER=${MYSQL_USER:-mysql} \
    MYSQL_UID=${MYSQL_UID:-116}

RUN \
##  Create the mysql system user \
##  ------------------------------------------------------------------------  ## \
    groupadd --system --force --gid "${MYSQL_UID}" "${MYSQL_USER}" \
    && useradd --system --uid "${MYSQL_UID}" --gid "${MYSQL_USER}" --shell /bin/bash --create-home "${MYSQL_USER}"

# add gosu for easy step-down from root
# https://github.com/tianon/gosu/releases
RUN set -eux \
	&& savedAptMark="$(apt-mark showmanual)" \
    && apt-get update -y \
	&& apt-get install -y --no-install-recommends \
        ca-certificates \
        dirmngr \
        gnupg \
        wget \
        vim \
        nano \
	&& dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')" \
	&& wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch" \
	&& wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch.asc" \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
	&& gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
	&& gpgconf --kill all \
	&& rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc \
	&& apt-mark auto '.*' > /dev/null \
	&& [ -z "$savedAptMark" ] || apt-mark manual $savedAptMark > /dev/null \
	&& chmod +x /usr/local/bin/gosu \
	&& gosu --version \
	&& gosu nobody true \
##  Clear cache \
##  -----------------------------------------------------------------------  ## \
    && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir /docker-entrypoint-initdb.d

RUN set -eux \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
		pwgen \
		openssl \
		perl \
##  Clear cache \
##  -----------------------------------------------------------------------  ## \
    && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

ARG MYSQL_DATA_DIR
ARG MYSQL_LOG_DIR
ARG MYSQL_RUN_DIR
ARG MYSQL_MAJOR
ARG MYSQL_VERSION
ARG DISTRO

ENV MYSQL_MAJOR=${MYSQL_MAJOR:-5.7} \
    MYSQL_VERSION=${MYSQL_VERSION:-5.7.31-1ubuntu16.04} \
    MYSQL_DATA_DIR=${MYSQL_DATA_DIR:-/var/lib/mysql} \
    MYSQL_RUN_DIR=${MYSQL_RUN_DIR:-/run/mysqld} \
    MYSQL_LOG_DIR=${MYSQL_LOG_DIR:-/var/log/mysql} \
    DISTRO=${DISTRO:-xenial}

## gpg: key 5072E1F5: public key "MySQL Release Engineering <mysql-build@oss.oracle.com>" imported \
##  ------------------------------------------------------------------------  ## \
RUN set -ex; \
    \
    key='A4A9406876FCBD3C456770C88C718D3B5072E1F5'; \
    export GNUPGHOME="$(mktemp -d)"; \
    gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
    gpg --export "$key" > /etc/apt/trusted.gpg.d/mysql.gpg; \
    rm -r "$GNUPGHOME"; \
    apt-key list > /dev/null \
    && echo "deb http://repo.mysql.com/apt/ubuntu/ ${DISTRO} mysql-${MYSQL_MAJOR}" > /etc/apt/sources.list.d/mysql.list

## the "/var/lib/mysql" stuff here is \
## because the mysql-server postinst doesn't have an explicit way \
## to disable the mysql_install_db codepath \
## besides having a database already "configured" (ie, stuff in /var/lib/mysql/mysql)
## also, we set debconf keys to make APT a little quieter
##  ------------------------------------------------------------------------  ## \
RUN { \
		echo mysql-community-server mysql-community-server/data-dir select ''; \
		echo mysql-community-server mysql-community-server/root-pass password ''; \
		echo mysql-community-server mysql-community-server/re-root-pass password ''; \
		echo mysql-community-server mysql-community-server/remove-test-db select false; \
	} | debconf-set-selections \
	&& apt-get update \
	&& apt-get install -y mysql-server="${MYSQL_VERSION}" \
	&& rm -rf /var/lib/mysql && mkdir -p /var/lib/mysql /var/run/mysqld \
	&& chown -R $MYSQL_USER:$MYSQL_USER /var/lib/mysql /var/run/mysqld \
## ensure that /var/run/mysqld (used for socket and lock files) is writable regardless of the UID our mysqld instance ends up having at runtime \
##  ------------------------------------------------------------------------  ## \
	&& chmod 777 /var/run/mysqld \
##  Clear cache \
##  -----------------------------------------------------------------------  ## \
    && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

## comment out a few problematic configuration values \
## don't reverse lookup hostnames, they are usually another container \
##  ------------------------------------------------------------------------  ## \
RUN sed -Ei 's/^(bind-address|log)/#&/' /etc/mysql/mysql.conf.d/mysqld.cnf \
	&& echo '[mysqld]\nskip-host-cache\nskip-name-resolve' > /etc/mysql/conf.d/docker.cnf

VOLUME /var/lib/mysql

COPY docker-entrypoint.sh /usr/local/bin/

RUN chmod +x /usr/local/bin/docker-*

## backwards compat \
##  ------------------------------------------------------------------------  ## \
RUN ln -s usr/local/bin/docker-entrypoint.sh /entrypoint.sh

ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 3306

CMD ["mysqld"]
