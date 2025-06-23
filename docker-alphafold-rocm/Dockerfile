# Copyright (C) Curii Corporation. All rights reserved.
#
# SPDX-License-Identifier: Apache-2.0

ARG ROCM_RELEASE=6.4.1
ARG PYTHON_VERSION=3.12
ARG ROCM_PATH="/opt/rocm-$ROCM_RELEASE"
ARG LLVM_PATH="$ROCM_PATH/llvm"
ARG AMDGPU_TARGETS="gfx1100"
ARG HHSUITE_PATH=/opt/hh-suite
ARG OPENMM_PATH=/opt/openmm

FROM rocm/tensorflow:rocm${ROCM_RELEASE}-py${PYTHON_VERSION}-tf2.18-dev AS build_base

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN DEBIAN_FRONTEND=noninteractive apt-get update --quiet && apt-get upgrade --yes --quiet \
    && apt-get install --no-install-recommends --yes --quiet \
        build-essential \
        git \
    && pip3 install -U pip 'setuptools<81'


FROM build_base AS build_jax

ARG ROCM_RELEASE
ARG ROCM_PATH
ARG LLVM_PATH

ARG ROCM_JAX_BRANCH=rocm-jaxlib-v0.6.0
RUN cd /root \
    && git clone https://github.com/ROCm/jax -b "$ROCM_JAX_BRANCH" \
    && cd jax \
    && python3 ./build/build.py build \
        --wheels=jaxlib,jax-rocm-plugin,jax-rocm-pjrt \
	--rocm_path="$ROCM_PATH" \
        --clang_path="$LLVM_PATH/bin/clang" \
    && python3 setup.py bdist_wheel


# Build Alphafold and related packages
FROM build_base AS build_alphafold

ARG ROCM_RELEASE
ARG ROCM_PATH
ARG LLVM_PATH

RUN DEBIAN_FRONTEND=noninteractive apt-get update --quiet && apt-get upgrade --yes --quiet \
    && apt-get install --no-install-recommends --yes --quiet \
        cmake \
	doxygen \
        tzdata \
        wget \
	curl

COPY ./pip-packages-rocm.txt /pip-packages.txt 
RUN pip3 install -r /pip-packages.txt

RUN mkdir -p /tmp/builds

# Compile HHsuite from source.
ARG HHSUITE_GIT_REFERENCE=bf3f42747ca4ddbd72e1c142aa3b648793c59045
ARG HHSUITE_PATH
RUN mkdir /tmp/builds/build-hh-suite \
    && git clone https://github.com/soedinglab/hh-suite.git /tmp/builds/hh-suite \
    && cd /tmp/builds/hh-suite \
    && git checkout -b hh-suite-dev "$HHSUITE_GIT_REFERENCE" \
    && cd /tmp/builds/build-hh-suite \
    && cmake /tmp/builds/hh-suite -DCMAKE_INSTALL_PREFIX="$HHSUITE_PATH" \
                                  -DCMAKE_BUILD_TYPE=Release \
    && make -j 16 \
    && make install

# Compile OpenMM from source.
ARG OPENMM_VERSION=8.0.0
ARG OPENMM_PATH
ARG PYTHON_VERSION
RUN mkdir -p /tmp/builds/build-openmm \
  && git clone https://github.com/openmm/openmm.git -b "$OPENMM_VERSION" /tmp/builds/openmm \
  && cd /tmp/builds/build-openmm \
  && cmake /tmp/builds/openmm -DCMAKE_C_COMPILER="$LLVM_PATH/bin/clang" \
                              -DCMAKE_CXX_COMPILER="$LLVM_PATH/bin/clang++" \
                              -DCMAKE_INSTALL_PREFIX="$OPENMM_PATH" \
                              -DOPENMM_BUILD_COMMON=ON \
                              -DOPENMM_PYTHON_USER_INSTALL=OFF \
                              -DPYTHON_EXECUTABLE="$(which python${PYTHON_VERSION})" \
                              -DCMAKE_BUILD_TYPE=Release \
  && make -j 16 \
  && make install \
  && make -j 16 PythonInstall

# Install OpenMM HIP plugin.
ARG OPENMM_HIP_GIT_REFERENCE=mi300_changes
ARG AMDGPU_TARGETS
RUN mkdir -p /tmp/builds/build-openmm-hip \
  && git clone https://github.com/amd/openmm-hip.git /tmp/builds/openmm-hip \
  && cd /tmp/builds/openmm-hip \
  && git checkout "$OPENMM_HIP_GIT_REFERENCE" \
  && cd /tmp/builds/build-openmm-hip \
  && cmake /tmp/builds/openmm-hip -DCMAKE_C_COMPILER="$LLVM_PATH/bin/clang" \
                                  -DCMAKE_CXX_COMPILER="$LLVM_PATH/bin/clang++" \
                                  -DOPENMM_DIR="$OPENMM_PATH" \
                                  -DOPENMM_SOURCE_DIR=/tmp/builds/openmm \
                                  -DCMAKE_INSTALL_PREFIX="$OPENMM_PATH" \
			          -DAMDGPU_TARGETS="$AMDGPU_TARGETS" \
                                  -DCMAKE_BUILD_TYPE=Release \
  && make -j 16 \
  && make install

# Install pdbfixer.
ARG PDBFIXER_GIT_REFERENCE=6bf10e138f6475f6f1850dae78d7bf1b420a1612
RUN git clone https://github.com/openmm/pdbfixer.git /tmp/builds/pdbfixer \
  && cd /tmp/builds/pdbfixer \
  && git checkout -b v1.9 "$PDBFIXER_GIT_REFERENCE" \
  && pip3 install .

# Clone alphafold 
ARG ALPHAFOLD_GIT_REFERENCE=020cd6d6cb16540114a084f9dbb8f21f811f9d21
ARG ALPHAFOLD_PATH=/app/alphafold
RUN git clone https://github.com/deepmind/alphafold "$ALPHAFOLD_PATH" \
  && cd "$ALPHAFOLD_PATH" \
  && git checkout -b alphafold-dev "$ALPHAFOLD_GIT_REFERENCE" \
  && sed -i 's#CUDA#HIP#g' alphafold/relax/amber_minimize.py \
  && cd "$ALPHAFOLD_PATH/alphafold/common" \
  && curl -LO https://git.scicore.unibas.ch/schwede/openstructure/-/raw/7102c63615b64735c4941278d92b554ec94415f8/modules/mol/alg/src/stereo_chemical_props.txt

# Install the re-built Jax wheels
RUN --mount=type=bind,from=build_jax,source=/root/jax/dist,target=/jax-wheels \
    pip3 install --no-deps --force-reinstall /jax-wheels/jax*.whl


FROM ubuntu:noble
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ARG ROCM_RELEASE
ARG OPENMM_PATH
ARG HHSUITE_PATH
ARG PYTHON_VERSION
ENV ROCM_PATH="/opt/rocm-$ROCM_RELEASE"
ENV LLVM_PATH="$ROCM_PATH/llvm"
ENV PATH="$OPENMM_PATH/bin:$HHSUITE_PATH/bin:$LLVM_PATH/bin:$PATH"

RUN DEBIAN_FRONTEND=noninteractive apt-get update --quiet \
    && apt-get upgrade --yes --quiet \
    && apt-get install --no-install-recommends --yes --quiet \
        hmmer \
        kalign \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get autoremove --yes \
    && apt-get clean \
    && ln -s "/usr/local/bin/python${PYTHON_VERSION}" /usr/bin/python3
COPY --from=build_alphafold /lib/x86_64-linux-gnu /lib/x86_64-linux-gnu
COPY --from=build_alphafold /usr/local /usr/local
COPY --from=build_alphafold /opt /opt
COPY --from=build_alphafold /etc/ld.so.cache /etc/
COPY --from=build_alphafold /app /app

WORKDIR /app/alphafold
