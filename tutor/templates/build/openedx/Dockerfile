FROM docker.io/ubuntu:16.04
MAINTAINER Overhang.io <contact@overhang.io>

############ common to lms & cms

# Install system requirements
RUN apt update && \
    # Global requirements
    apt install -y language-pack-en git python-virtualenv build-essential software-properties-common curl git-core libmysqlclient-dev libxml2-dev libxslt1-dev python3 python3-virtualenv python3-apt python3-dev libxmlsec1-dev libfreetype6-dev swig gcc g++ \
    # openedx requirements
    gettext gfortran graphviz graphviz-dev libffi-dev libfreetype6-dev libgeos-dev libjpeg8-dev liblapack-dev libpng12-dev libsqlite3-dev libxml2-dev libxmlsec1-dev libxslt1-dev lynx nodejs npm ntp pkg-config \
    && rm -rf /var/lib/apt/lists/*

# Dockerize will be useful to wait for mysql DB availability
ARG DOCKERIZE_VERSION=v0.6.1
RUN curl -L -o /tmp/dockerize.tar.gz https://github.com/jwilder/dockerize/releases/download/$DOCKERIZE_VERSION/dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz \
    && tar -C /usr/local/bin -xzvf /tmp/dockerize.tar.gz \
    && rm /tmp/dockerize.tar.gz

# Checkout edx-platform code: https://github.com/edx/edx-platform/commits/open-release/ironwood.master
# Because we are pulling from a branch, and not a tag, you should manually modify the
# date hash below to force clearing the docker cache.
ARG EDX_PLATFORM_REPOSITORY=https://github.com/edx/edx-platform.git
ARG EDX_PLATFORM_VERSION=open-release/juniper.alpha1
ARG EDX_PLATFORM_VERSION_DATE=20200227
RUN mkdir -p /openedx/edx-platform && \
    echo "Puling $EDX_PLATFORM_VERSION tag from $EDX_PLATFORM_REPOSITORY ($EDX_PLATFORM_VERSION_DATE)" && \
    git clone $EDX_PLATFORM_REPOSITORY --branch $EDX_PLATFORM_VERSION --depth 1 /openedx/edx-platform
WORKDIR /openedx/edx-platform

# Download extra locales to /openedx/locale
RUN cd /tmp \
    && curl -L -o openedx-i18n.tar.gz https://github.com/openedx/openedx-i18n/archive/ironwood.tar.gz \
    && tar xzf /tmp/openedx-i18n.tar.gz \
    && mv openedx-i18n-ironwood/edx-platform/locale/ /openedx/locale/ \
    && rm -rf openedx-i18n*

# TODO move this higher up
ENV LC_ALL en_US.UTF-8

# Install python requirements in a virtualenv
RUN python3 -m virtualenv --python=python3.5 /openedx/venv
ENV PATH /openedx/venv/bin:${PATH}
ENV VIRTUAL_ENV /openedx/venv/
RUN pip install setuptools==39.0.1 pip==19.3.1
RUN pip install -r requirements/edx/base.txt

# Install a recent version of nodejs
RUN nodeenv /openedx/nodeenv --node=12.13.0 --prebuilt
ENV PATH /openedx/nodeenv/bin:${PATH}

# Install nodejs requirements
RUN npm set progress=false \
    && npm install
ENV PATH ./node_modules/.bin:${PATH}

# Install private requirements: this is useful for installing custom xblocks.
COPY ./requirements/ /openedx/requirements
RUN cd /openedx/requirements/ \
  && touch ./private.txt \
  && pip install -r ./private.txt

# Create folder that will store lms/cms.env.json files, as well as
# the tutor-specific settings files.
RUN mkdir -p /openedx/config ./lms/envs/tutor ./cms/envs/tutor
COPY revisions.yml /openedx/config/
ENV LMS_CFG /openedx/config/lms.env.json
ENV STUDIO_CFG /openedx/config/cms.env.json
ENV REVISION_CFG /openedx/config/revisions.yml
COPY settings/lms/*.py ./lms/envs/tutor/
COPY settings/cms/*.py ./cms/envs/tutor/

# Copy scripts
COPY ./bin /openedx/bin
RUN chmod a+x /openedx/bin/*
ENV PATH /openedx/bin:${PATH}

{{ patch("openedx-dockerfile-pre-assets") }}

# Collect production assets. By default, only assets from the default theme
# will be processed. This makes the docker image lighter and faster to build.
# Only the custom themes added to /openedx/themes will be compiled.
# Here, we don't run "paver update_assets" which is slow, compiles all themes
# and requires a complex settings file. Instead, we decompose the commands
# and run each one individually to collect the production static assets to
# /openedx/staticfiles.
ENV NO_PYTHON_UNINSTALL 1
RUN openedx-assets xmodule \
    && openedx-assets npm \
    && openedx-assets webpack --env=prod \
    && openedx-assets common
COPY ./themes/ /openedx/themes/
RUN openedx-assets themes \
    && openedx-assets collect --settings=tutor.assets

# Create a data directory, which might be used (or not)
RUN mkdir /openedx/data

# service variant is "lms" or "cms"
ENV SERVICE_VARIANT lms
ENV SETTINGS tutor.production

{{ patch("openedx-dockerfile") }}

# Entrypoint will set right environment variables
ENTRYPOINT ["docker-entrypoint.sh"]

# Run server
COPY gunicorn_conf.py /openedx/gunicorn_conf.py
EXPOSE 8000
CMD gunicorn -c /openedx/gunicorn_conf.py --name ${SERVICE_VARIANT} --bind=0.0.0.0:8000 --max-requests=1000 --access-logfile - ${SERVICE_VARIANT}.wsgi:application
