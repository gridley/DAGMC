# Svalinn DAGMC Dockerfile

# build from repository root directory with command
# docker build -t dagmc -f CI/Dockerfile .

# Global ARGS set before the first build stage are accessable by all build stages
ARG EMBREE_BRANCH='v3.6.1'
ARG geant4_version=11.1.2
ARG UBUNTU_VERSION=20.04
ARG MOAB_BRANCH=5.3.0
ARG double_down=OFF
ARG ci_jobs=4
ARG HDF5_VERSION=1.14.3

ARG build_dir=/root/build_dir
ARG install_dir=/root/opt

ARG COMPILER=gcc

FROM ubuntu:${UBUNTU_VERSION} AS base

# Use bash as the default shell
SHELL ["/bin/bash", "-c"]

# timezone setup
ENV TZ=America/Chicago
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# Update core packages
RUN apt-get -y update; \
    apt-get -y install autoconf \
                       clang \
                       cmake \
                       g++ \
                       gcc \
                       gfortran \
                       libhdf5-dev \
                       libtool \
                       libeigen3-dev\
                       python3-numpy \
                       python3 \
                       python3-pip \
                       python3-setuptools \
                       python3-dev \
                       libpython3-dev \
                       wget \
                       software-properties-common; \
    add-apt-repository ppa:git-core/ppa; \
    apt-get update; \
    apt-get install -y git; \
    update-alternatives --install /usr/bin/python python /usr/bin/python3 10; \
    update-alternatives --install /usr/bin/pip pip /usr/bin/pip3 10; \
    pip install "cython<3";

ARG build_dir
ARG install_dir

ENV build_dir=${build_dir}
ENV install_dir=${install_dir}

FROM base as compiler-gcc

ENV CC=gcc
ENV CXX=g++

FROM base as compiler-clang

ENV CC=clang
ENV CXX=clang++

FROM compiler-${COMPILER} as external_deps

# accessing gloabl ARGs in build stage
ARG geant4_version
ARG EMBREE_BRANCH
ARG install_dir 
ARG build_dir
ARG ci_jobs
ARG CXX
ARG CC

ENV geant4_basename=geant4-v${geant4_version}
ENV geant4_tarball=${geant4_basename}.tar.gz
ENV geant4_shasum=2397eb859dc4de095ff66059d8bda9f060fdc42e10469dd7890946293eeb0e39
ENV geant4_build_dir=${build_dir}/geant4
ENV geant4_install_dir=${install_dir}/geant4

RUN mkdir -p ${geant4_build_dir}/build && \
    cd ${geant4_build_dir} && \
    wget https://gitlab.cern.ch/geant4/geant4/-/archive/v${geant4_version}/${geant4_tarball} --no-check-certificate && \
    tar -xzf ${geant4_tarball} && \
    cd build && \
    cmake ../${geant4_basename} -DCMAKE_INSTALL_RPATH=${geant4_install_dir}/lib \
                                -DCMAKE_INSTALL_PREFIX=${geant4_install_dir} \
                                -DGEANT4_USE_SYSTEM_EXPAT=OFF \
                                -DCMAKE_BUILD_TYPE=Release \
                                -DCMAKE_CXX_COMPILER=${CXX} \
                                -DCMAKE_C_COMPILER=${CC} \
                                -DBUILD_STATIC_LIBS=ON && \
    make -j${ci_jobs} && \
    make install && \
    cd && \
    rm -rf ${geant4_build_dir}


ENV embree_install_dir=${install_dir}/embree
ENV embree_build_dir=${build_dir}/embree

# Clone and install Embree
RUN mkdir -p ${embree_build_dir}/build && \
    cd ${embree_build_dir} && \
    git clone -b ${EMBREE_BRANCH} https://github.com/embree/embree && \
    cd build && \
    cmake ../embree -DCMAKE_INSTALL_PREFIX=${embree_install_dir} \
                    -DEMBREE_TASKING_SYSTEM=INTERNAL \
                    -DEMBREE_ISPC_SUPPORT=OFF \
                    -DEMBREE_TUTORIALS=OFF \
                    -DEMBREE_TBB_ROOT=/usr && \
    make -j${ci_jobs} && \
    make -j${ci_jobs} install && \
    cd && \
    rm -rf ${embree_build_dir}


FROM external_deps AS hdf5

# accessing gloabl ARGs in build stage
ARG HDF5_VERSION
ARG install_dir
ARG build_dir
ARG ci_jobs
ARG CXX
ARG CC

ENV hdf5_build_dir=${build_dir}/hdf5
ENV hdf5_install_dir=${install_dir}/hdf5

RUN mkdir -p ${hdf5_build_dir}/build && \
    cd ${hdf5_build_dir} && \
    export HDF5_VERSION_major=`python -c "print('.'.join('${HDF5_VERSION}'.split('.')[:-1]))"` && \
    wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-${HDF5_VERSION_major}/hdf5-${HDF5_VERSION}/src/hdf5-${HDF5_VERSION}.tar.gz && \
    tar -xzf hdf5-${HDF5_VERSION}.tar.gz && \
    cd build && \
    ../hdf5-${HDF5_VERSION}/configure --enable-shared \
        --prefix=${hdf5_install_dir} \
        CXX=${CXX} \
        CC=${CC} && \
    make -j${ci_jobs} && \
    make install && \
    cd && \
    rm -rf ${hdf5_build_dir}


FROM hdf5 AS moab

# accessing gloabl ARGs in build stage
ARG install_dir
ARG MOAB_BRANCH
ARG build_dir
ARG ci_jobs
ARG CXX
ARG CC


# Set MOAB env variable
ENV moab_build_dir=${build_dir}/moab
ENV moab_install_dir=${install_dir}/moab


RUN mkdir -p ${moab_build_dir}/build && \
    cd ${moab_build_dir} && \
    git clone -b ${MOAB_BRANCH} --depth 1 https://bitbucket.org/fathomteam/moab && \
    cd build && \
    cmake ../moab -DCMAKE_INSTALL_RPATH=${hdf5_install_dir}/lib:${moab_install_dir}/lib \
                  -DCMAKE_BUILD_TYPE=Release \
                  -DENABLE_HDF5=ON -DHDF5_ROOT=${hdf5_install_dir} \
                  -DCMAKE_INSTALL_PREFIX=${moab_install_dir} \
                  -DCMAKE_CXX_COMPILER=${CXX} \
                  -DCMAKE_C_COMPILER=${CC} \
                  -DENABLE_BLASLAPACK=OFF \
                  -DBUILD_SHARED_LIBS=ON \
                  -DENABLE_FORTRAN=OFF \
                  -DENABLE_PYMOAB=ON && \
    make -j${ci_jobs} && \
    make install && \
    cd && \
    rm -rf ${moab_build_dir}

ENV double_down_build_dir=${build_dir}/double-down
ENV double_down_install_dir=${install_dir}/double-down

# Clone and install Double-Down
# performed after moab install as double-down requires moab
RUN mkdir -p ${double_down_build_dir}/build && \
    cd ${double_down_build_dir} && \
    git clone --shallow-submodules --single-branch --branch v1.0.0 --depth 1 https://github.com/pshriwise/double-down.git && \
    cd build && \
    cmake ../double-down -DMOAB_DIR=${moab_install_dir} \
                         -DCMAKE_INSTALL_PREFIX=${double_down_install_dir} \
                         -DEMBREE_DIR=${embree_install_dir} && \
    make -j${ci_jobs} && \
    make -j${ci_jobs} install


FROM moab as dagmc

# accessing gloabl ARGs in build stage
ARG install_dir
ARG build_dir
ARG double_down
ARG CXX
ARG CC

ENV dagmc_build_dir=${build_dir}/dagmc
ENV dagmc_install_dir=${install_dir}/dagmc

# copies the enitre git repo into the dockerfile to ensure the submodules can also be found
RUN mkdir -p ${dagmc_build_dir}/build
COPY . ${dagmc_build_dir}
RUN cd  ${dagmc_build_dir} && git submodule update --init

RUN mkdir -p ${dagmc_build_dir}/build && \
    cd ${dagmc_build_dir} && \
    cd build && \
    cmake ../ -DMOAB_DIR=${moab_install_dir} \
              -DBUILD_GEANT4=ON \
              -DGEANT4_DIR=${geant4_install_dir} \
              -DBUILD_CI_TESTS=ON \
              # reg tests disabled see issue https://github.com/svalinn/DAGMC/issues/858
              -DBUILD_MW_REG_TESTS=OFF \
              -DBUILD_STATIC_EXE=OFF \
              -DBUILD_STATIC_LIBS=OFF \
              -DCMAKE_C_COMPILER=${CC} \
              -DCMAKE_CXX_COMPILER=${CXX} \
              -DCMAKE_Fortran_COMPILER=gfortran \
              -DCMAKE_INSTALL_PREFIX=${dagmc_install_dir} \
              -DDOUBLE_DOWN=${double_down} \
              -Ddd_ROOT=${double_down_install_dir} && \
    make -j${ci_jobs} && \
    make install


FROM dagmc as dagmc_test

ENV dagmc_build_dir=${build_dir}/dagmc
ENV dagmc_install_dir=${install_dir}/dagmc

# Test DAGMC executables
RUN cd ${dagmc_build_dir}/build && \
    PATH=${dagmc_install_dir}/bin:$PATH CTEST_OUTPUT_ON_FAILURE=1 make test

# clean out config test directory for next build
RUN cd ${dagmc_build_dir} && \
    git clean -dxf .

# Test DAGMC CMake configuration file
RUN cd ${dagmc_build_dir}/cmake/test_config && \
    cmake . -DDAGMC_ROOT=${dagmc_install_dir} && \
    CTEST_OUTPUT_ON_FAILURE=1 make all test

