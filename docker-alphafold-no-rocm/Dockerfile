# Copyright (C) Curii Corporation. All rights reserved.
#
# SPDX-License-Identifier: Apache-2.0

ARG HHSUITE_PATH=/opt/hh-suite
ARG OPENMM_PATH=/opt/openmm

FROM ubuntu:noble AS build

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ARG PYTHON_VERSION=3.12

RUN DEBIAN_FRONTEND=noninteractive apt-get update --quiet \
    && apt-get upgrade --yes --quiet \
    && apt-get install --no-install-recommends --yes --quiet \
        build-essential \
        cmake \
	curl \
	doxygen \
        git \
	python3-pip \
	python3-dev \
	swig \
        tzdata \
        wget

# Update pip and Python packages.
COPY pip-packages.txt /pip-packages.txt
RUN pip3 install --break-system-packages -r /pip-packages.txt

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
  && cmake /tmp/builds/openmm \ 
                           -DCMAKE_INSTALL_PREFIX="$OPENMM_PATH" \
                           -DOPENMM_BUILD_COMMON=ON \
                           -DOPENMM_PYTHON_USER_INSTALL=OFF \
                           -DPYTHON_EXECUTABLE="$(which "python${PYTHON_VERSION}")" \
                           -DCMAKE_BUILD_TYPE=Release \
  && make -j 16 \
  && make install \
  && make -j 16 PythonInstall

# Install pdbfixer.
ARG PDBFIXER_GIT_REFERENCE=6bf10e138f6475f6f1850dae78d7bf1b420a1612
RUN git clone https://github.com/openmm/pdbfixer.git /tmp/builds/pdbfixer \
  && cd /tmp/builds/pdbfixer \
  && git checkout -b v1.9 "$PDBFIXER_GIT_REFERENCE" \
  && pip3 install --break-system-packages .

# Clone alphafold 
ARG ALPHAFOLD_GIT_REFERENCE=020cd6d6cb16540114a084f9dbb8f21f811f9d21
ARG ALPHAFOLD_PATH=/app/alphafold
RUN git clone https://github.com/deepmind/alphafold "$ALPHAFOLD_PATH" \
  && cd "$ALPHAFOLD_PATH" \
  && git checkout -b alphafold-dev "$ALPHAFOLD_GIT_REFERENCE" \
  && cd "$ALPHAFOLD_PATH/alphafold/common" \
  && curl -LO https://git.scicore.unibas.ch/schwede/openstructure/-/raw/7102c63615b64735c4941278d92b554ec94415f8/modules/mol/alg/src/stereo_chemical_props.txt

FROM ubuntu:noble
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ARG OPENMM_PATH
ARG HHSUITE_PATH
ENV CUDA_VISIBLE_DEVICES=-1
ENV PATH="$OPENMM_PATH/bin:$HHSUITE_PATH/bin:$PATH"

RUN DEBIAN_FRONTEND=noninteractive apt-get update --quiet \
    && apt-get upgrade --yes --quiet \
    && apt-get install --no-install-recommends --yes --quiet \
	python3 \
        hmmer \
        kalign \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get autoremove --yes \
    && apt-get clean

COPY --from=build /usr/local /usr/local
COPY --from=build /opt /opt
COPY --from=build /etc/ld.so.cache /etc/
COPY --from=build /app /app

WORKDIR /app/alphafold
