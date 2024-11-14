FROM python:3.12-alpine

LABEL authors="UKDS-Jisc"
# Tag passed through via the Makefile
ARG CKAN_VERSION=${CKAN_VERSION}

ENV TZ="Europe/London"
ENV APP_DIR=/srv/app
ENV SRC_DIR=${APP_DIR}/src
ENV CKAN_INI=${APP_DIR}/ckan.ini
ENV PIP_SRC=${SRC_DIR}
ENV CKAN_STORAGE_PATH=/var/lib/ckan
ENV GIT_URL=https://github.com/ckan/ckan.git

# Customize these in the environment (.env) file if needed
ENV CKAN_SITE_URL=http://localhost:5000
ENV CKAN__PLUGINS="image_view text_view datatables_view datastore envvars"

# UWSGI options
ENV UWSGI_HARAKIRI=50

WORKDIR ${APP_DIR}

# Set up timezone
RUN apk --no-cache add tzdata && \
    ln -s /usr/share/zoneinfo/${TZ} /etc/localtime

# Install necessary packages to run CKAN
RUN apk add --no-cache curl \
        g++ \
        gettext \
        git \
        libmagic \
        libxml2 \
        libxslt \
        musl-dev \
        patch \
        postgresql-client \
        py3-gevent \
        py3-pip \
        uwsgi \
        uwsgi-corerouter \
        uwsgi-gevent \
        uwsgi-http \
        uwsgi-python && \
    # Packages to build CKAN requirements and plugins
    apk add --no-cache --virtual .build-deps \
        autoconf \
        automake \
        cargo \
        gcc \
        libffi-dev \
        libtool \
        libxml2-dev \
        libxslt-dev \
        linux-headers \
        make \
        openssl-dev \
        postgresql-dev

# Create the src directory
RUN mkdir -p ${SRC_DIR}

# Install uwsgi, the CKAN application, the dependency packages for CKAN plus some confiquration
RUN pip3 install -U pip && \
    pip3 install uwsgi && \
    cd ${SRC_DIR} && \
    pip3 install -e git+${GIT_URL}@${CKAN_VERSION}#egg=ckan && \
    pip3 install --no-binary markdown -r ckan/requirements.txt && \
    # Install CKAN envvars to support loading config from environment variables
    pip3 install -e git+https://github.com/okfn/ckanext-envvars.git@v0.0.6#egg=ckanext-envvars && \
    # Install CKAN Plugins
    pip3 install ckanext-pdfview \
            ckanext-geoview \
            ckanext-matomo \
            ckanext-xloader && \
    pip3 install -r 'https://raw.githubusercontent.com/ckan/ckanext-xloader/master/requirements.txt' && \
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

# Create a local user and group plus set up the storage path
RUN addgroup -g 92 ckan && \
    adduser -u 92 -h /srv/app -s /bin/bash -D -G ckan ckan && \
    mkdir -p ${CKAN_STORAGE_PATH} && \
    chown -R ckan:ckan ${CKAN_STORAGE_PATH}

COPY setup/prerun.py ${APP_DIR}
COPY setup/start_ckan.sh ${APP_DIR}
ADD https://raw.githubusercontent.com/ckan/ckan/${CKAN_VERSION}/wsgi.py ${APP_DIR}
RUN chmod 644 ${APP_DIR}/wsgi.py

# Create entrypoint directory for children image scripts
ONBUILD RUN mkdir /docker-entrypoint.d

EXPOSE 5000

CMD ["/srv/app/start_ckan.sh"]
