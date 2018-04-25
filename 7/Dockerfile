FROM drush/drush:8 as intermediate

ARG PLATFORM=pfizer
ARG GIT_USER=pfizer
ARG DEBUG=n

ENV PLATFORM_BRANCH=7.x-3.x-dev \
    PLATFORM_REPO="platform-${PLATFORM}"

RUN echo "Update apt-get and install git."; \
    apt-get -qq update && apt-get -qq install git; \
    echo "Finished updating apt-get and installing git."; \
    rm -rf /var/lib/apt/lists/*; \
    mkdir /root/.ssh; \
    touch /root/.ssh/known_hosts;

COPY .ssh /root/.ssh

RUN chmod 400 -R /root/.ssh

# Can only be qwh, hcp or pfizer
ENV MAKE_FILE="drupal.make"

RUN if [ "${PLATFORM}" = "qwh" ]; then \
      export PLATFORM_BRANCH='7.x-1.x-dev'; \
    elif [ "${PLATFORM}" = "hcp" ]; then \
      export PLATFORM_BRANCH='7.x-hcp-2.10.x-dev'; \
    else \
      echo "--build-arg PLATFORM MUST either be qwh, hcp or pfizer (default)."; \
      echo "Other args are not accepted. Falling back to default (pfizer) so that build can continue."; \
      PLATFORM=pfizer; \
    fi; \
    echo "The following platform will now be built: platform-${PLATFORM}-${PLATFORM_BRANCH}"; \
    export PLATFORM_REPO="platform-${PLATFORM}"; \
    \
    rm -rf /web/*; \
    \
    mkdir /git /web; \
    cd /git; \
    ssh-keyscan github.com >> /root/.ssh/known_hosts; \
    eval "$(ssh-agent -s)"; \
    ssh-add /root/.ssh/id_rsa; \
    echo git@github.com:${GIT_USER}/${PLATFORM_REPO}.git; \
    git clone git@github.com:${GIT_USER}/${PLATFORM_REPO}.git; \
    cd /git/${PLATFORM_REPO}; \
    git fetch --all; \
    git checkout ${PLATFORM_BRANCH}; \
    git pull; \
    \
    cp /git/${PLATFORM_REPO}/${MAKE_FILE} /web; \
    cd /web; \
    if [ "${DEBUG}" = "y" ]; then \
      echo "We dont run drush make during debug - This helps to save time."; \
      less ${MAKE_FILE}; \
      ls -al; \
      drush status; \
    else \
      drush make ${MAKE_FILE} --force-complete -y; \
      tar -zcvf ${FILE} /web; \
    fi; \
    echo "Cache killer: 0"

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
    \
    DRUSH_PATCHFILE_URL="https://bitbucket.org/davereid/drush-patchfile.git"

USER root

RUN mkdir -p ${WODBY_DIR_FILES} /var/www/builds

USER www-data

# Delete then copy files from intermediate.
RUN rm -rf ${DOC_ROOT}/* && \
    mkdir ${DOC_ROOT}

COPY --from=intermediate /web ${DOC_ROOT}

USER root

RUN set -ex; \
    \
    # Drush
    su-exec www-data composer global require drush/drush:^8; \
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
    \
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
    su-exec www-data composer clear-cache; \
    su-exec www-data drush cc drush

USER www-data

COPY templates /etc/gotpl/
COPY actions /usr/local/bin
COPY init /docker-entrypoint-init.d/
