# Copyright (C) Curii Corporation. All rights reserved.
#
# SPDX-License-Identifier: Apache-2.0

FROM rocm/tensorflow:rocm6.4-py3.10-tf2.18-dev

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ARG AMDGPU_TARGETS=gfx1100

ARG ROCM_RELEASE=6.4.0
ENV ROCM_PATH="/opt/rocm-$ROCM_RELEASE"
ENV LLVM_PATH="$ROCM_PATH/llvm"

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update --quiet && apt-get upgrade --yes --quiet
RUN apt-get install --no-install-recommends --yes --quiet \
        build-essential \
        cmake \
	doxygen \
        git \
        hmmer \
        kalign \
        tzdata \
        wget \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get autoremove --yes \
    && apt-get clean

# Update pip and Python packages.
COPY pip-packages-rocm.txt /pip-packages.txt
RUN pip3 install -U pip && pip3 install -r /pip-packages.txt

RUN mkdir -p /tmp/builds

# Compile HHsuite from source.
ARG HHSUITE_VERSION=3.3.0
ARG HHSUITE_PATH=/opt/hh-suite
RUN mkdir /tmp/builds/build-hh-suite \
   && git clone --branch "v$HHSUITE_VERSION" --single-branch https://github.com/soedinglab/hh-suite.git /tmp/builds/hh-suite \
    && cd /tmp/builds/build-hh-suite \
    && cmake /tmp/builds/hh-suite -DCMAKE_INSTALL_PREFIX="$HHSUITE_PATH" \
                                  -DCMAKE_BUILD_TYPE=Release \
    && make -j 8 \
    && make install

# Compile OpenMM from source.
ARG OPENMM_VERSION=8.0.0
ARG OPENMM_PATH=/opt/openmm
RUN mkdir -p /tmp/builds/build-openmm \
  && git clone https://github.com/openmm/openmm.git -b "$OPENMM_VERSION" /tmp/builds/openmm \
  && cd /tmp/builds/build-openmm \
  && cmake /tmp/builds/openmm -DCMAKE_C_COMPILER="$LLVM_PATH/bin/clang" \
                              -DCMAKE_CXX_COMPILER="$LLVM_PATH/bin/clang++" \
                              -DCMAKE_INSTALL_PREFIX="$OPENMM_PATH" \
                              -DOPENMM_BUILD_COMMON=ON \
                              -DOPENMM_PYTHON_USER_INSTALL=OFF \
                              -DPYTHON_EXECUTABLE="$(which python3)" \
                              -DCMAKE_BUILD_TYPE=Release \
  && make -j 8 \
  && make install \
  && make -j 8 PythonInstall

# Install OpenMM HIP plugin.
ARG OPENMM_HIP_GIT_REFERENCE=mi300_changes
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
  && make -j 8 \
  && make install

ENV PATH="$OPENMM_PATH/bin:$HHSUITE_PATH/bin:$LLVM_PATH/bin:$PATH"

# Install pdbfixer.
ARG PDBFIXER_GIT_REFERENCE=6bf10e138f6475f6f1850dae78d7bf1b420a1612
RUN git clone https://github.com/openmm/pdbfixer.git /opt/builds/pdbfixer \
  && cd /opt/builds/pdbfixer \
  && git checkout -b v1.9 "$PDBFIXER_GIT_REFERENCE" \
  && pip3 install .

RUN cd / && rm -rf /tmp/builds

# Clone alphafold 
ARG ALPHAFOLD_GIT_REFERENCE=020cd6d6cb16540114a084f9dbb8f21f811f9d21
ARG ALPHAFOLD_PATH=/app/alphafold
RUN git clone https://github.com/deepmind/alphafold "$ALPHAFOLD_PATH" \
  && cd "$ALPHAFOLD_PATH" \
  && git checkout -b alphafold-dev "$ALPHAFOLD_GIT_REFERENCE" \
  && sed -i 's#CUDA#HIP#g' alphafold/relax/amber_minimize.py \
  && cd "$ALPHAFOLD_PATH/alphafold/common" \
  && curl -LO https://git.scicore.unibas.ch/schwede/openstructure/-/raw/7102c63615b64735c4941278d92b554ec94415f8/modules/mol/alg/src/stereo_chemical_props.txt

WORKDIR /app/alphafold
