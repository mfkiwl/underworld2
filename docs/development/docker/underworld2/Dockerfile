#####################################################################
# Multi stage Dockerfile structure:
# 1. runtime 
# 2. build
# 3. final == runtime + min. build
#
# It begins with layers for runtime execution. 
# The runtime environment (packages, permissions, ENV vars.) 
# are consistent accross all layer of this Dockerfile. 
# The build layer takes the runtime layer and builds the software
# stack in /usr/local.
# The final image is a composite of the runtime layer and 
# minimal sections of the build layer.
#####################################################################

# The following are passed in via --build-args
# Used for github actions on the underworld repo
# Must go before the 1st FROM see
# https://docs.docker.com/engine/reference/builder/#understand-how-arg-and-from-interact
ARG PETSC_IMAGE="underworldcode/petsc:3.22.2-ompi"
ARG LAVAVU_IMAGE="none"
ARG PYTHON_VERSION="3.12"
ARG BASE_IMAGE="python:$PYTHON_VERSION-slim-bookworm"

# 'petsc-image' will be used later on in build stage COPY command 
FROM ${PETSC_IMAGE} as petsc-image

FROM ${LAVAVU_IMAGE} as lavavu-image

FROM ${BASE_IMAGE} as runtime
LABEL maintainer="https://github.com/underworldcode/"

# need to repeat ARGS after every FROM 
ARG PYTHON_VERSION

#### Dockerfile ENV vars - for all image stages
ENV LANG=C.UTF-8
# mpi lib will be install at /usr/local/lib
ENV LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH

# python env vars. 
# prepappending on PATH means all pip install will goto the PYOPT
ENV PYVER=${PYTHON_VERSION}
ENV PYOPT=/opt/venv
ENV PATH=$PYOPT/bin:$PATH
ENV PYTHONPATH=$PYTHONPATH:$PYOPT/lib/python${PYVER}/site-packages
# taken from PETSC images
ENV PETSC_DIR=/usr/local
ENV PYTHONPATH=$PYTHONPATH:$PETSC_DIR/lib

# add user jovyan
ENV NB_USER jovyan
ENV NB_HOME /home/$NB_USER
RUN useradd -m -s /bin/bash -N $NB_USER

# runtime packages
RUN apt-get update -qq \
&&  DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends \
        ca-certificates \
        bash-completion \
        ssh \
        libopenblas0 \
        libxml2 \
        vim \
        git \
        # python3-venv \
        # python3-dev \
        # python3-full \
        # libpng16-16t64 \
        # libjpeg8 \
        # libavcodec60 \
        # libavformat60 \
        # libavutil58 \
        # libswscale7 \
        libpng16-16 \
        libjpeg-dev \
        libtiff-dev \ 
        libglu1-mesa \
        libosmesa6 \
        libavcodec-dev \
        libavformat-dev \
        libavutil-dev \
        libswscale-dev \ 
        zlib1g \
&&  apt-get clean \
&&  rm -rf /var/lib/apt/lists/*

# start a virtual environment install runtime modules in 'build' as some
# packages need gcc
RUN python3 -m venv $PYOPT \
&&  chmod ugo+rwx $PYOPT

FROM runtime as build

ARG PYTHON_VERSION

# install build packages
RUN apt-get update -qq \
&&  DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends \
        swig \
        cmake \
        make \
        gfortran \
        pkg-config \
        libxml2-dev \
        ninja-build \
&&  apt-get clean \
&&  rm -rf /var/lib/apt/lists/*

# Remove this for future versions
# setuptools=65.6.0 has a unfixed error, so forcing version
RUN pip3 install setuptools --force-reinstall --no-cache \
&&  pip3 install --no-cache-dir \
         matplotlib \
         "cython==3.0.10" \
         "scipy==1.13" \
         "numpy<2" \
         pint \
         rise \
         pytest \
         nbmake \
         gFlex \
         tabulate \
         jupytext \
         lavavu

#### Only install lavavu via pip. Much more straightforward then separate image copy
## Lets grab lavavu & requirements
#COPY --from=lavavu-image /opt       /opt
#COPY --from=lavavu-image /usr/local /usr/local
#RUN apt-get update \
#&&  DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends $(awk '{print $1'} /opt/installed.txt) \
#&&  apt-get clean \
#&&  rm -rf /var/lib/apt/lists/* 
#RUN pip3 install -r /opt/requirements.txt

# mpi, petsc, mpi4py, petsc4py, h5py
COPY --from=petsc-image /opt       /opt
COPY --from=petsc-image /usr/local /usr/local
RUN apt-get update \
&&  DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends $(awk '{print $1'} /opt/installed.txt) \
&&  apt-get clean \
&&  rm -rf /var/lib/apt/lists/*
RUN pip3 install -r /opt/requirements.txt

# install h5py with MPI enabled
RUN CC=mpicc HDF5_MPI="ON" HDF5_DIR=${PETSC_DIR} pip3 install --no-cache-dir --no-build-isolation --no-binary=h5py h5py \
&& pip install --no-cache-dir jupyterlab

# for development, uncomment the following lines and comment out `git clone`
# WORKDIR /tmp
# COPY --chown=$NB_USER:users . /tmp/underworld2
RUN git clone --depth 1 --branch v2.16.x https://github.com/underworldcode/underworld2.git /tmp/underworld2
WORKDIR /tmp/underworld2
RUN pip3 install -vvv .
# TODO: Use offical badlands release made - not github with pip.
RUN pip3 install --no-cache-dir \
       "badlands @ git+https://github.com/badlands-model/badlands.git#subdirectory=badlands"
# "git+https://github.com/drufat/triangle.git" 

RUN pip3 freeze >/opt/requirements.txt
# Record manually install apt packages.
RUN apt-mark showmanual >/opt/installed.txt


# Build the final image, a combination of the runtime and build stages
FROM runtime as final

ARG PYTHON_VERSION

COPY --from=build --chown=$NB_USER:users /opt /opt
COPY --from=build --chown=$NB_USER:users /usr/local /usr/local

# must make directory before COPY into it for permissions to work ()
# set default viewer to a notebook see https://jupytext.readthedocs.io/en/latest/text-notebooks.html#with-a-double-click
RUN mkdir -p $NB_HOME/workspace $NB_HOME/Underworld/UWGeodynamics \
&&  chown $NB_USER:users -R $NB_HOME \
&& jupyter-server extension enable --sys-prefix jupyter_server_proxy \
&& jupytext-config set-default-viewer 

#Copy in examples, tests, etc.
COPY --chown=$NB_USER:users ./docs/examples   $NB_HOME/Underworld/examples
COPY --chown=$NB_USER:users ./docs/cheatsheet $NB_HOME/Underworld/cheatsheet
COPY --chown=$NB_USER:users ./docs/user_guide $NB_HOME/Underworld/user_guide
COPY --chown=$NB_USER:users ./docs/test       $NB_HOME/Underworld/test
COPY --chown=$NB_USER:users ./docs/UWGeodynamics/examples  $NB_HOME/Underworld/UWGeodynamics/examples
COPY --chown=$NB_USER:users ./docs/UWGeodynamics/benchmarks  $NB_HOME/Underworld/UWGeodynamics/benchmarks
COPY --chown=$NB_USER:users ./docs/UWGeodynamics/tutorials  $NB_HOME/Underworld/UWGeodynamics/tutorials

EXPOSE 8888
WORKDIR $NB_HOME
USER $NB_USER

# Declare a volume space
VOLUME $NB_HOME/workspace

CMD ["jupyter-lab", "--no-browser", "--ip='0.0.0.0'"]
