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
    PLATFORM_PFIZER_BRANCH="7.x-3.x-dev" \
    PLATFORM_HCP2_REPO="platform-hcp" \
    PLATFORM_HCP2_BRANCH="7.x-hcp-2.10.x-dev" \
    PLATFORM_QWH_REPO="platform-qwh" \
    PLATFORM_QWH_BRANCH="7.x-1.x-dev" \
    MAKE_FILE="drupal.make"

ENV \
    PLATFORM_REPO=${PLATFORM_QWH_REPO} \
    PLATFORM_BRANCH=${PLATFORM_QWH_BRANCH}
#    PLATFORM_REPO=${PLATFORM_PFIZER_REPO} \
#    PLATFORM_BRANCH=${PLATFORM_PFIZER_BRANCH}
#    PLATFORM_REPO=${PLATFORM_HCP2_REPO} \
#    PLATFORM_BRANCH=${PLATFORM_HCP2_BRANCH}

RUN rm -rf /web/* && \
    mkdir /git /web && \
    cd /git && \
    ssh-keyscan github.com >> /root/.ssh/known_hosts && \
    eval "$(ssh-agent -s)" && \
    ssh-add /root/.ssh/id_rsa && \
    ls -al && \
    echo git@github.com:${DEV_GIT_USER}/${PLATFORM_REPO}.git && \
    git clone git@github.com:${DEV_GIT_USER}/${PLATFORM_REPO}.git && \
    cd /git/${PLATFORM_REPO} && \
    ls /git/${PLATFORM_REPO} && \
    git fetch --all && \
    git checkout ${PLATFORM_BRANCH} && \
    ls /git/${PLATFORM_REPO} && \
    git pull && \
    cp /git/${PLATFORM_REPO}/${MAKE_FILE} /web && \
    cd /web && \
    less ${MAKE_FILE} && \
    ls -al && \
    drush status && \
    drush make ${MAKE_FILE} --force-complete -y && \
    echo "Cache killer: 11"

#ARG buildtag
ENV FROM_TAG=7.1-3.0.0

FROM wodby/php:7.1-3.0.0

ENV DRUSH_LAUNCHER_VER="0.4.3" \
    \
    WODBY_DIR_FILES="/mnt/files" \
    WODBY_DIR_CONF="/var/www/conf" \
    \
    PROJECT_FILES="/var/www/html" \
    DOC_ROOT="/var/www/html/web" \
    \
    PHP_REALPATH_CACHE_TTL="3600" \
    PHP_OUTPUT_BUFFERING="16384" \

    DRUSH_PATCHFILE_URL="https://bitbucket.org/davereid/drush-patchfile.git"

USER root

RUN mkdir ${WODBY_DIR_FILES}

USER www-data

# Delete then copy files from intermediate.
RUN rm -rf ${DOC_ROOT}/* && \
    mkdir ${DOC_ROOT}

COPY --from=intermediate /web ${DOC_ROOT}

USER root

RUN set -ex; \
    \
    # Drush
    su-exec www-data composer global require drush/drush:8; \
    \
    # Drush launcher
    wget -O drush.phar \
        "https://github.com/drush-ops/drush-launcher/releases/download/${DRUSH_LAUNCHER_VER}/drush.phar"; \
    chmod +x drush.phar; \
    mv drush.phar /usr/local/bin/drush; \
    \
    # Drush extensions
    su-exec www-data drush @none dl registry_rebuild-7.x; \
    su-exec www-data git clone "${DRUSH_PATCHFILE_URL}" /home/www-data/.drush/drush-patchfile; \

    # Drupal console
    curl https://drupalconsole.com/installer -L -o drupal.phar; \
    mv drupal.phar /usr/local/bin/drupal; \
    chmod +x /usr/local/bin/drupal; \
    \
    mv /usr/local/bin/actions.mk /usr/local/bin/php.mk; \
    # Change overridden target name to avoid warnings.
    sed -i 's/git-checkout:/php-git-checkout:/' /usr/local/bin/php.mk; \
    \
    mkdir -p "${WODBY_DIR_FILES}" "${WODBY_DIR_CONF}"; \
    chown www-data:www-data "${WODBY_DIR_CONF}"; \
    \
    # Clean up
    su-exec www-data composer clear-cache; \
    su-exec www-data drush cc drush

USER www-data

COPY templates /etc/gotpl/
COPY actions /usr/local/bin
COPY init /docker-entrypoint-init.d/