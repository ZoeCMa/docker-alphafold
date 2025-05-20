# Copyright (C) Curii Corporation. All rights reserved.
#
# SPDX-License-Identifier: Apache-2.0

FROM ubuntu:jammy-20250404

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update --quiet && apt-get upgrade --yes --quiet
RUN apt-get install --no-install-recommends --yes --quiet \
        build-essential \
        cmake \
	curl \
	doxygen \
        git \
        hmmer \
        kalign \
	python3-pip \
	python3-dev \
	swig \
        tzdata \
        wget \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get autoremove --yes \
    && apt-get clean

# Update pip and Python packages.
COPY pip-packages.txt /pip-packages.txt
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
  && cmake /tmp/builds/openmm \ 
                           -DCMAKE_INSTALL_PREFIX="$OPENMM_PATH" \
                           -DOPENMM_BUILD_COMMON=ON \
                           -DOPENMM_PYTHON_USER_INSTALL=OFF \
                           -DPYTHON_EXECUTABLE="$(which python3)" \
                           -DCMAKE_BUILD_TYPE=Release \
  && make -j 8 \
  && make install \
  && make -j 8 PythonInstall

ENV PATH="$OPENMM_PATH/bin:$HHSUITE_PATH/bin:$PATH"

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
  && cd "$ALPHAFOLD_PATH/alphafold/common" \
  && curl -LO https://git.scicore.unibas.ch/schwede/openstructure/-/raw/7102c63615b64735c4941278d92b554ec94415f8/modules/mol/alg/src/stereo_chemical_props.txt

ENV CUDA_VISIBLE_DEVICES=-1
WORKDIR /app/alphafold
