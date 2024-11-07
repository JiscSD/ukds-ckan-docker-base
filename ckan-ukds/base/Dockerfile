FROM python:3.13-alpine

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
RUN apk --no-cache add tzdata && ln -s /usr/share/zoneinfo/${TZ} /etc/localtime

# Install system libraries
RUN apk update && apk --no-cache add -y \
        apk-tools \
        g++ \
        git \
        libpq-dev \
        libtool \
        linux-headers-generic \
        wget

# Create the src directory
RUN mkdir -p ${SRC_DIR}

# Install uwsgi, the CKAN application, the dependency packages for CKAN plus some confiquration
RUN pip3 install -U pip && \
    pip3 install uwsgi && \
    cd ${SRC_DIR} && \
    pip3 install -e git+${GIT_URL}@${CKAN_VERSION}#egg=ckan && \
    cd ckan && \
    pip3 install --no-binary markdown -r requirements.txt && \
    # Install CKAN envvars to support loading config from environment variables
    pip3 install -e git+https://github.com/okfn/ckanext-envvars.git@v0.0.6#egg=ckanext-envvars && \
    # Install CKAN Plugins \
    pip3 install ckanext-pdfview \
            ckanext-geoview \
            ckanext-harvest \
            ckanext-matomo \
            ckanext-xloader && \
    pip3 install -r 'https://raw.githubusercontent.com/ckan/ckanext-xloader/master/requirements.txt' && \
    pip3 install -e 'git+https://github.com/ckan/ckanext-pages.git#egg=ckanext-pages' && \
    pip3 install -U requests[security] && \
    git clone https://github.com/JiscSD/ckanext-customised_fields_from_tag_vocabulary.git && \
    cd ckanext-customised_fields_from_tag_vocabulary && \
    pip3 install -e . && \
    pip3 install -r requirements.txt && \
    cd .. && \
    git clone https://github.com/JiscSD/ckanext-ukdstheme.git && \
    cd ckanext-ukdstheme && \
    pip install -e . && \
    pip install -r requirements.txt && \
    cd .. && \
    git clone https://github.com/JiscSD/ckanext-dataset_geo_display.git && \
    cd ckanext-dataset_geo_display && \
    pip install -e . && \
    pip install -r requirements.txt && \
    cd .. && \
    # Create and update CKAN config
    ckan generate config ${CKAN_INI} && \
    ckan config-tool ${CKAN_INI} "SECRET_KEY = " && \
    ckan config-tool ${CKAN_INI} "ckan.plugins = ${CKAN__PLUGINS}"

# Create a local user and group plus set up the storage path
RUN groupadd -g 92 ckan && \
    useradd -rm -d /srv/app -s /bin/bash -g ckan -u 92 ckan && \
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
