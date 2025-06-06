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

ARG MPI_IMAGE="underworldcode/openmpi:4.1.4"
ARG PYTHON_VERSION="3.12"
ARG BASE_IMAGE="python:$PYTHON_VERSION-slim-bookworm"

FROM ${MPI_IMAGE} as mpi-image

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
ENV PETSC_DIR=/usr/local
ENV PYTHONPATH=$PYTHONPATH:$PETSC_DIR/lib

# add user jovyan
ENV NB_USER jovyan
ENV NB_HOME /home/$NB_USER
RUN useradd -m -s /bin/bash -N $NB_USER

RUN apt-get update -qq \
&&  DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends \
        bash-completion \
        ssh \
        libopenblas0 \
        # python3-full \
        # python3-dev \
&& apt-get clean \
&& rm -rf /var/lib/apt/lists/*

# build and open virtual environment
RUN python3 -m venv $PYOPT \
&&  chmod ugo+rwx $PYOPT \
&&  pip3 install -U setuptools \
                    wheel

FROM runtime as build

ARG PYTHON_VERSION

RUN apt-get update -qq \
&&  DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends \
        ca-certificates \
        wget \
        build-essential \
        gfortran \
        g++ \
        cmake \
        libopenblas-dev \
        libz-dev \
        git \
&&  apt-get clean \
&&  rm -rf /var/lib/apt/lists/*

# copy in from mpi container
COPY --from=mpi-image /opt/installed.txt /opt/installed.txt
COPY --from=mpi-image /usr/local /usr/local
RUN apt-get update -qq \
&&  DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends $(awk '{print $1'} /opt/installed.txt) \
&&  apt-get clean \
&&  rm -rf /var/lib/apt/lists/*

RUN pip3 install --no-cache-dir \
      "cython==3.0.10" \
      "numpy<2" \
      mpi4py

# get petsc
ARG PETSC_VERSION="3.22.2"
RUN mkdir -p /tmp/src
WORKDIR /tmp/src
RUN wget https://web.cels.anl.gov/projects/petsc/download/release-snapshots/petsc-lite-${PETSC_VERSION}.tar.gz --no-check-certificate \
&& tar -zxf petsc-lite-${PETSC_VERSION}.tar.gz 
WORKDIR /tmp/src/petsc-${PETSC_VERSION}              

# install petsc as root
RUN PETSC_DIR=`pwd` ./configure --with-debugging=0 --prefix=/usr/local \
                --COPTFLAGS="-g -O3" --CXXOPTFLAGS="-g -O3" --FOPTFLAGS="-g -O3" \
                --with-petsc4py=1               \
                --with-zlib=1                   \
                --with-shared-libraries=1       \
                --with-cxx-dialect=C++11        \
                --with-make-np=4                \
                --download-hdf5=https://github.com/HDFGroup/hdf5/archive/refs/tags/hdf5-1_10_8.tar.gz \
                --download-mumps=1              \
                --download-parmetis=1           \
                --download-metis=1              \
                --download-superlu=1            \
                --download-hypre=1              \
                --download-scalapack=1          \
                --download-superlu_dist=1       \
                --download-pragmatic=1          \
                --download-ctetgen              \
                --download-eigen                \
                --download-superlu=1            \
                --download-triangle             \
                --useThreads=0                  \
&&  make PETSC_DIR=`pwd` PETSC_ARCH=arch-linux-c-opt all \
&&  make PETSC_DIR=`pwd` PETSC_ARCH=arch-linux-c-opt install \
&&  rm -rf /usr/local/share/petsc

### move h5py install to UW script
# # install h5py with MPI enabled
# RUN CC=mpicc HDF5_MPI="ON" HDF5_DIR=${PETSC_DIR} pip3 install --no-cache-dir --no-build-isolation --no-binary=h5py h5py \
# && pip install --no-cache-dir jupyterlab

# record builder stage packages used
RUN pip3 freeze > /opt/requirements.txt \
&&  apt-mark showmanual > /opt/packages.txt


FROM runtime as minimal

COPY --from=build /opt /opt
COPY --from=build /usr/local /usr/local

# switch to not-root user and workspace
USER $NB_USER
WORKDIR $NB_HOME

# default command is to run jupyter lab
CMD ["jupyter-lab", "--no-browser", "--ip='0.0.0.0'"]
