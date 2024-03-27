##################
### Build CKAN ###
##################
FROM alpine:3.14.2 as ckanbuild

# Used by Github Actions to tag the image with
ENV IMAGE_TAG=2.8.12

# Set CKAN version to build
ENV GIT_URL=https://github.com/ckan/ckan.git
ENV GIT_BRANCH=ckan-2.8.12

# Set src dirs
ENV SRC_DIR=/srv/app/src
ENV PIP_SRC=${SRC_DIR}

WORKDIR ${SRC_DIR}

# Packages to build CKAN requirements and plugins
RUN apk add --no-cache \
        git \
        curl \
        python2 \
        postgresql-dev \
        linux-headers \
        gcc \
        make \
        g++ \
        autoconf \
        automake \
        patch \
        libtool \
        musl-dev \
        pcre-dev \
        pcre \
        python2-dev \
        libffi-dev \
        libxml2-dev \
        libxslt-dev

# Create the src directory
RUN mkdir -p ${SRC_DIR}

# Install pip
RUN curl -o ${SRC_DIR}/get-pip.py https://bootstrap.pypa.io/pip/2.7/get-pip.py && \
    python ${SRC_DIR}/get-pip.py 'pip==20.3.3'

# Fetch and build CKAN and requirements
RUN pip install -e git+${GIT_URL}@${GIT_BRANCH}#egg=ckan
# Copy patches and apply patches script
COPY ./patches ${SRC_DIR}/patches
COPY ./scripts/apply_ckan_patches.sh ${SRC_DIR}/apply_ckan_patches.sh
# Apply patches
RUN ${SRC_DIR}/apply_ckan_patches.sh
RUN rm -rf /srv/app/src/ckan/.git
RUN pip wheel --wheel-dir=/wheels -r ckan/requirements.txt
RUN pip wheel --wheel-dir=/wheels uwsgi==2.0.19.1 gevent==21.12.0 greenlet==1.1.3


###########################
### Default-Extensions ####
###########################
FROM alpine:3.14.2 as extbuild

# Set src dirs
ENV SRC_DIR=/srv/app/src
ENV PIP_SRC=${SRC_DIR}

# List of default extensions
ENV DEFAULT_EXTENSIONS envvars

# Locations and tags, please use specific tags or revisions
ENV ENVVARS_GIT_URL=https://github.com/okfn/ckanext-envvars
ENV ENVVARS_GIT_BRANCH=0.0.1
ENV HARVEST_GIT_URL=https://github.com/ckan/ckanext-harvest
ENV HARVEST_GIT_BRANCH=v1.5.6
ENV DCAT_GIT_URL=https://github.com/ckan/ckanext-dcat
ENV DCAT_GIT_BRANCH=v1.5.1
ENV OAUTH2_GIT_URL=https://gitlab.com/urban-dataspace-platform/use_cases/ckan/ckanext-oauth2-py3
ENV OAUTH2_GIT_BRANCH=v0.7.0

RUN apk add --no-cache \
        git \
        curl \
        python2 \
        python2-dev

# Create the src directory
RUN mkdir -p ${SRC_DIR}

# Install pip
RUN curl -o ${SRC_DIR}/get-pip.py https://bootstrap.pypa.io/pip/2.7/get-pip.py && \
    python ${SRC_DIR}/get-pip.py 'pip==20.3.3'

# Fetch and build the default CKAN extensions
RUN pip wheel --wheel-dir=/wheels git+${ENVVARS_GIT_URL}@${ENVVARS_GIT_BRANCH}#egg=ckanext-envvars
#CKAN Harvester
RUN pip wheel --wheel-dir=/wheels git+${HARVEST_GIT_URL}@${HARVEST_GIT_BRANCH}#egg=ckanext-harvest
RUN pip wheel --wheel-dir=/wheels -r https://raw.githubusercontent.com/ckan/ckanext-harvest/${HARVEST_GIT_BRANCH}/requirements.txt
RUN curl -o /wheels/harvest.txt https://raw.githubusercontent.com/ckan/ckanext-harvest/${HARVEST_GIT_BRANCH}/requirements.txt
#CKAN DCAT extension
RUN pip wheel --wheel-dir=/wheels git+${DCAT_GIT_URL}@${DCAT_GIT_BRANCH}#egg=ckanext-dcat
RUN pip wheel --wheel-dir=/wheels -r https://raw.githubusercontent.com/ckan/ckanext-dcat/${DCAT_GIT_BRANCH}/requirements.txt
RUN curl -o /wheels/dcat.txt https://raw.githubusercontent.com/ckan/ckanext-dcat/${DCAT_GIT_BRANCH}/requirements.txt
#CKAN OAUTH2 extension
RUN pip wheel --wheel-dir=/wheels git+${OAUTH2_GIT_URL}@${OAUTH2_GIT_BRANCH}#egg=ckanext-oauth2
RUN pip wheel --wheel-dir=/wheels -r https://gitlab.com/urban-dataspace-platform/use_cases/ckan/ckanext-oauth2-py3/-/raw/${OAUTH2_GIT_BRANCH}/requirements/install.pip
RUN curl -o /wheels/oauth2.txt https://gitlab.com/urban-dataspace-platform/use_cases/ckan/ckanext-oauth2-py3/-/raw/${OAUTH2_GIT_BRANCH}/requirements/install.pip

############
### MAIN ###
############
FROM alpine:3.14.2

LABEL maintainer="Keitaro Inc <info@keitaro.com>"
LABEL org.opencontainers.image.source https://github.com/keitaroinc/docker-ckan

ENV APP_DIR=/srv/app
ENV SRC_DIR=/srv/app/src
ENV CKAN_DIR=${SRC_DIR}/ckan
ENV DATA_DIR=/srv/app/data
ENV PIP_SRC=${SRC_DIR}
ENV CKAN_SITE_URL=http://localhost:5000
ENV CKAN__PLUGINS oauth2 envvars image_view text_view recline_view datastore datapusher harvest ckan_harvester dcat dcat_rdf_harvester dcat_json_harvester dcat_json_interface structured_data 

# Install necessary packages to run CKAN
RUN apk add --no-cache git \
        bash \
        git \
        gettext \
        curl \
        postgresql-client \
        python2 \
        libmagic \
        pcre \
        libxslt \
        libxml2 \
        tzdata \
        apache2-utils && \
    # Create SRC_DIR
    mkdir -p ${SRC_DIR}

# Install pip
RUN curl -o ${SRC_DIR}/get-pip.py https://bootstrap.pypa.io/pip/2.7/get-pip.py && \
    python ${SRC_DIR}/get-pip.py 'pip==20.3.3'

# Get artifacts from build stages
COPY --from=ckanbuild /wheels /srv/app/wheels
COPY --from=extbuild /wheels /srv/app/ext_wheels
COPY --from=ckanbuild /srv/app/src/ckan ${CKAN_DIR}

# Additional install steps for build stages artifacts
RUN pip install --no-index --find-links=/srv/app/wheels uwsgi==2.0.19.1 gevent==21.12.0

# Create a local user and group to run the app
RUN addgroup -g 92 -S ckan && \
    adduser -u 92 -h /srv/app -H -D -S -G ckan ckan

WORKDIR ${CKAN_DIR}

# Install CKAN
RUN pip install -e /srv/app/src/ckan && \
    cp who.ini ${APP_DIR} && \
    pip install --no-index --find-links=/srv/app/wheels -r requirements.txt && \
    # Install default CKAN extensions
    pip install --no-index --find-links=/srv/app/ext_wheels ckanext-envvars && \
	#CKAN harvester
    pip install --no-index --find-links=/srv/app/ext_wheels ckanext-harvest && \
    pip install --no-index --find-links=/srv/app/ext_wheels -r /srv/app/ext_wheels/harvest.txt && \
    #CKAN DCAT
    pip install --no-index --find-links=/srv/app/ext_wheels ckanext-dcat && \
    pip install --no-index --find-links=/srv/app/ext_wheels -r /srv/app/ext_wheels/dcat.txt && \
	#CKAN OAUTH2
    pip install --no-index --find-links=/srv/app/ext_wheels ckanext-oauth2 && \
    pip install --no-index --find-links=/srv/app/ext_wheels -r /srv/app/ext_wheels/oauth2.txt && \
    # Create and update CKAN config
    # Set timezone
    echo "UTC" >  /etc/timezone && \
    # Generate CKAN config
    paster --plugin=ckan make-config ckan ${APP_DIR}/production.ini && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "beaker.session.secret = " && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.plugins = ${CKAN__PLUGINS}" && \
	# Configure CKAN OAUTH2
	paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.oauth2.register_url = https://YOUR_OAUTH_SERVICE/users/sign_up" && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.oauth2.reset_url = https://YOUR_OAUTH_SERVICE/users/password/new" && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.oauth2.edit_url = https://YOUR_OAUTH_SERVICE/settings" && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.oauth2.authorization_endpoint = https://mytutilo2.ddns.net/realms/myrealm/protocol/openid-connect/auth" && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.oauth2.token_endpoint = https://mytutilo2.ddns.net/realms/myrealm/protocol/openid-connect/token" && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.oauth2.profile_api_url = https://YOUR_OAUTH_SERVICE/user" && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.oauth2.client_id = myclient" && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.oauth2.client_secret = o23cJkWDBVONaZUUfVBULvB7PG12Si9S" && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.oauth2.scope = profile openid email" && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.oauth2.rememberer_name = auth_tkt" && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.oauth2.profile_api_user_field = JSON_FIELD_TO_FIND_THE_USER_IDENTIFIER" && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.oauth2.profile_api_fullname_field = JSON_FIELD_TO_FIND_THE_USER_FULLNAME" && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.oauth2.profile_api_mail_field = JSON_FIELD_TO_FIND_THE_USER_MAIL" && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.oauth2.authorization_header = OAUTH2_HEADER" && \
    # Set the default level for extensions to INFO
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini -s logger_ckanext -e level=INFO && \
    # Create the data directory
    mkdir ${DATA_DIR} && \
    # Change ownership to app user
    chown -R ckan:ckan /srv/app

# Remove wheels
RUN rm -rf /srv/app/wheels /srv/app/ext_wheels

# Install python2 secrets for generating sessions
RUN pip install python2-secrets 

# Copy necessary scripts
COPY setup/app ${APP_DIR}

WORKDIR ${APP_DIR}

# Copy the alias script for paster to be ckan so it's compatible with 2.9
COPY setup/bin/ckan /usr/bin/ckan

# Create entrypoint directory for children image scripts
ONBUILD RUN mkdir docker-entrypoint.d

# Create afterinit directory for children image scripts
ONBUILD RUN mkdir docker-afterinit.d

EXPOSE 5000

HEALTHCHECK --interval=10s --timeout=5s --retries=5 CMD curl --fail http://localhost:5000/api/3/action/status_show || exit 1

USER ckan

CMD ["/srv/app/start_ckan.sh"]
