FROM drush/drush:8 as intermediate

RUN apt-get update && apt-get install git && \
    rm -rf /var/lib/apt/lists/* && \
    mkdir /root/.ssh && \
    touch /root/.ssh/known_hosts && \
    echo "Listing /root" && \
    ls -al /

COPY .ssh /root/.ssh

RUN chmod 400 -R /root/.ssh

ENV PFIZER_GIT_USER="pfizer" \
    DEV_GIT_USER="unipro-ArnaudKalicharam" \
    PLATFORM_PFIZER_REPO="platform-pfizer" \
    PLATFORM_BRANCH="7.x-3.x-dev" \
    PLATFORM_QWH_REPO="platform-qwh" \
    PLATFORM_QWH_BRANCH="7.x-qwh-1.0.x-dev" \
    MAKE_FILE="drupal.make"

RUN mkdir /git /web && \
    cd /git && \
    ssh-keyscan github.com >> /root/.ssh/known_hosts && \
    eval "$(ssh-agent -s)" && \
    ssh-add /root/.ssh/id_rsa && \
    ls -al && \
    git clone git@github.com:${DEV_GIT_USER}/${PLATFORM_QWH_REPO}.git && \
    cd /git/${PLATFORM_QWH_REPO} && \
    ls /git/${PLATFORM_QWH_REPO} && \
    git fetch --all && \
    git checkout ${PLATFORM_QWH_BRANCH} && \
    ls /git/${PLATFORM_QWH_REPO} && \
    git pull && \
    cp /git/${PLATFORM_QWH_REPO}/${MAKE_FILE} /web

RUN cd /web && \
    ls -al && \
    drush status && \
    drush make ${MAKE_FILE} --force-complete -y

RUN echo "Dont forget to bring all changes to this makefile to other dockerfiles."

FROM wodby/php:5.3

ENV WODBY_DIR_FILES="/mnt/files" \
    WODBY_DIR_CONF="/var/www/conf" \

    PHP_ALWAYS_POPULATE_RAW_POST_DATA="-1" \
    PHP_MBSTRING_ENCODING_TRANSLATION="Off" \
    PHP_MBSTRING_HTTP_INPUT="pass" \
    PHP_MBSTRING_HTTP_OUTPUT="pass" \
    PHP_OUTPUT_BUFFERING="16384" \
    PHP_REALPATH_CACHE_SIZE="64k" \
    PHP_REALPATH_CACHE_TTL="3600" \
    PHP_SESSION_AUTO_START="0" \

    DRUSH_PATCHFILE_URL="https://bitbucket.org/davereid/drush-patchfile.git"

USER root

RUN set -ex && \

    # Drush
    su-exec www-data composer global require drush/drush:7.4.0 && \

    # Drush extensions
    su-exec www-data drush @none dl registry_rebuild-7.x && \
    su-exec www-data git clone "${DRUSH_PATCHFILE_URL}" /home/www-data/.drush/drush-patchfile && \

    mv /usr/local/bin/actions.mk /usr/local/bin/php.mk && \
    # Change overridden target name to avoid warnings.
    sed -i 's/git-checkout:/php-git-checkout:/' /usr/local/bin/php.mk && \

    mkdir -p "${WODBY_DIR_FILES}" "${WODBY_DIR_CONF}" && \
    chown www-data:www-data "${WODBY_DIR_CONF}" && \

    # Clean up
    su-exec www-data composer clear-cache && \
    su-exec www-data drush cc drush

USER www-data

COPY templates /etc/gotpl/
COPY actions /usr/local/bin
COPY init /docker-entrypoint-init.d/