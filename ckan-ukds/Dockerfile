ARG ENV=base

FROM python:3.12-alpine AS python

# ┌─────────────────────────────────────────────────────────────┐
# │                                                             │
# │ Base image (Production env)                                 │
# │ ---------------------------                                 │
# │ This is always built                                        │
# │                                                             │
# └─────────────────────────────────────────────────────────────┘

FROM python AS base

LABEL authors="UKDS-Jisc"

# Tag passed through via the Makefile
ARG CKAN_REF=${CKAN_REF}
# Make it available to child images
ENV CKAN_REF=${CKAN_REF}

# Internals, you probably don't need to change these
ENV TZ="Europe/London"
ENV APP_DIR=/srv/app
ENV SRC_DIR=${APP_DIR}/src
ENV CKAN_INI=${APP_DIR}/ckan.ini
ENV PIP_SRC=${SRC_DIR}
ENV CKAN_STORAGE_PATH=/var/lib/ckan
ENV GIT_URL=https://github.com/ckan/ckan.git
ENV CKAN_INSTALL_VERSION=2.11
ENV CKAN_SYS_USER=ckan_sys
ENV CKAN_USER=ckan

# Customize these in the environment (.env) file if needed
ENV CKAN_SITE_URL=http://localhost:5000
ENV CKAN__PLUGINS="image_view text_view datatables_view datastore envvars"

# UWSGI options
ENV UWSGI_HARAKIRI=50

WORKDIR ${APP_DIR}

# Set up timezone
RUN apk --no-cache add tzdata && \
    ln -s /usr/share/zoneinfo/${TZ} /etc/localtime

# Set LC_ALL=en_GB.UTF-8 will ensure that all locale-dependent operations in the current environment
# will use English language and United States cultural conventions with UTF-8 character encoding
ENV LC_ALL=en_GB.UTF-8

# Install necessary packages to run CKAN
RUN apk add --no-cache apk-tools \
        curl \
        g++ \
        gcc \
        git \
        libmagic \
        libpq-dev \
        libxml2 \
        libxslt \
        --upgrade linux-headers \
        musl \
        musl-utils \
        patch \
        wget

# Create the src directory
RUN mkdir -p ${SRC_DIR}

# Install uwsgi, the CKAN application, the dependency packages for CKAN plus some confiquration
RUN pip3 install -U pip && \
    pip3 install uwsgi && \
    cd ${SRC_DIR} && \
    pip3 install -e git+${GIT_URL}@${CKAN_INSTALL_VERSION}#egg=ckan && \
    pip3 install --no-binary markdown -r ckan/requirements.txt && \
    # Install CKAN envvars to support loading config from environment variables
    pip3 install -e git+https://github.com/okfn/ckanext-envvars.git@v0.0.6#egg=ckanext-envvars && \
    # Install CKAN Plugins
    pip3 install ckanext-pdfview \
            ckanext-envvars \
            ckanext-geoview \
            ckanext-matomo \
            ckanext-xloader && \
    pip3 install -r https://raw.githubusercontent.com/ckan/ckanext-xloader/master/requirements.txt && \
    pip3 install -e git+https://github.com/ckan/ckanext-pages.git#egg=ckanext-pages && \
    pip3 install -U requests[security] && \
    pip3 install -e git+https://github.com/ckan/ckanext-harvest.git#egg=ckanext-harvest && \
    pip3 install -r ckanext-harvest/requirements.txt && \
    pip3 install -e git+https://github.com/ckan/ckanext-dcat.git#egg=ckanext-dcat && \
    pip3 install -r ckanext-dcat/requirements.txt && \
    git clone https://github.com/JiscSD/ckanext-customised_fields_from_tag_vocabulary.git && \
    pip3 install -e ckanext-customised_fields_from_tag_vocabulary/. && \
    pip3 install -r ckanext-customised_fields_from_tag_vocabulary/requirements.txt && \
    git clone https://github.com/JiscSD/ckanext-ukdstheme.git && \
    pip3 install -e ckanext-ukdstheme/. && \
    pip3 install -r ckanext-ukdstheme/requirements.txt && \
    git clone https://github.com/JiscSD/ckanext-dataset_geo_display.git && \
    pip install -e ckanext-dataset_geo_display/. && \
    pip install -r ckanext-dataset_geo_display/requirements.txt && \
    # Create and update CKAN config
    ckan generate config ${CKAN_INI} && \
    ckan config-tool ${CKAN_INI} "SECRET_KEY = " && \
    ckan config-tool ${CKAN_INI} "ckan.plugins = ${CKAN__PLUGINS}"

# Create ckan and ckan-sys users and the ckan-sys group plus set up the storage path
RUN addgroup -g 502 ${CKAN_SYS_USER} && \
    adduser -S -h /srv/app -s /bin/bash -G ${CKAN_SYS_USER} -u 502 ${CKAN_SYS_USER} && \
    adduser -S -h /srv/app -s /bin/bash -G ${CKAN_SYS_USER} -u 503 ${CKAN_USER}

COPY setup/prerun.py ${APP_DIR}
COPY setup/start_ckan.sh ${APP_DIR}
ADD https://raw.githubusercontent.com/ckan/ckan/${CKAN_INSTALL_VERSION}/wsgi.py ${APP_DIR}
RUN chmod 644 ${APP_DIR}/wsgi.py

# Create entrypoint directory for children image scripts
RUN mkdir -p /docker-entrypoint.d && chmod 755 /docker-entrypoint.d

# Set the ownership of the app directory, usr/local and the entrypoint directory to the ${CKAN_SYS_USER} user
RUN chown -R ${CKAN_SYS_USER}:${CKAN_SYS_USER} ${APP_DIR} && \
    chown -R ${CKAN_SYS_USER}:${CKAN_SYS_USER} /docker-entrypoint.d && \
    chown -R ${CKAN_SYS_USER}:${CKAN_SYS_USER} /usr/local

# Set the ownership of the CKAN config file, src and the storage path to the ckan user
RUN chown ckan:${CKAN_SYS_USER} ${APP_DIR}/ckan.ini && \
    chown -R ckan:${CKAN_SYS_USER} ${APP_DIR}/src && \
    mkdir -p ${CKAN_STORAGE_PATH} && \
    chown -R ckan:${CKAN_SYS_USER} ${CKAN_STORAGE_PATH}

USER ${CKAN_USER}

EXPOSE 5000

CMD ["/srv/app/start_ckan.sh"]

# ┌─────────────────────────────────────────────────────────────┐
# │                                                             │
# │ Dev image (Development env)                                 │
# │ ---------------------------                                 │
# │ This is only built when ENV=dev                             │
# │                                                             │
# └─────────────────────────────────────────────────────────────┘

FROM base AS dev

ENV SRC_EXTENSIONS_DIR=${APP_DIR}/src_extensions

USER root

RUN cd ${SRC_DIR}/ckan && \
pip3 install -r https://raw.githubusercontent.com/ckan/ckan/${CKAN_INSTALL_VERSION}/dev-requirements.txt

COPY --chown=${CKAN_SYS_USER}:${CKAN_SYS_USER} setup/unsafe.cert setup/unsafe.key setup/start_ckan_development.sh setup/install_src.sh ${APP_DIR}

# Update local directories
RUN mkdir -p ${SRC_EXTENSIONS_DIR} /var/lib/ckan && \
    chown -R ${CKAN_SYS_USER}:${CKAN_SYS_USER} ${SRC_EXTENSIONS_DIR} && \
    chown -R ckan:${CKAN_SYS_USER} /var/lib/ckan/ && \
    chmod 775 ${SRC_EXTENSIONS_DIR}

USER ${CKAN_USER}

CMD ["/srv/app/start_ckan_development.sh"]

# ──────────────────────────────────────────────────────────────

FROM ${ENV} AS final