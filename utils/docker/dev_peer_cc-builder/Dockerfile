# Copyright IBM Corp. All Rights Reserved.
# Copyright 2020 Intel Corporation
#
# SPDX-License-Identifier: Apache-2.0

# Description:
#   Sets up fabric/fpc source environment and builds, in different stages, three different
#   containers
#   - peer: a container with an fpc-enhanced fabric peer
#   - cc-builder: an environment to build fpc chaincode enclave.so
#   - dev: a fpc-enabled interactive development environment
#
#  Configuration (build) paramaters (for defaults, see below section with ARGs)
#  - fpc image version:         FPC_VERSION
#  - fabric repo:               FABRIC_REPO
#  - fabric branch:             FABRIC_VERSION
#  - fpc repo:                  FPC_REPO_URL
#  - fpc branch/tag/commit      FPC_REPO_BRANCH_TAG_OR_COMMIT
#  - git user:                  GIT_USER_NAME
#  - git user's email:          GIT_USER_EMAIL
#  - sgxmode:                   SGX_MODE
#  - sgx attestation mode:      FPC_ATTESTATION_TYPE
#  - additional dev apt pkgs:   APT_ADD_DEV_PKGS

# Note on SGX_MODE:
# In a docker build environment, we build but cannot _run_ with SGX_MODE=HW!.
# Moreoever, libraries like sgxssl are be agnostic to the mode in which they are built and
# can be used in both modes. Hence, even when we define glabally SGX_MODE to be HW,
# we can still build _and test_ them by locally/temporarily defining SGX_MODE to SIM
# as we do below for sgxssl ...

# global config params
ARG FPC_VERSION=latest
ARG SGX_MODE=SIM
ARG FPC_ATTESTATION_TYPE
ARG APT_DEV_ADD_PKGS=

# global constants
# paths relative to GOPATH. to share across stages, has to be defined global but at
# this stage we do not know GOPATH, so has to be relative ...
ARG FABRIC_REL_PATH=src/github.com/hyperledger/fabric
ARG FPC_REL_PATH=src/github.com/hyperledger-labs/fabric-private-chaincode


# Note on multi-stage: pre-docker build kit, all stages are evaluated, even if they are not required.
# While our default setting is to use build kit (see ../../config.mk), we order here in decreasing
# importance of how regularly we need the builds so builds are also faster without build-kit

FROM hyperledger/fabric-private-chaincode-base-dev:${FPC_VERSION} as common

# import global vars
ARG FABRIC_REL_PATH
ARG FPC_REL_PATH
ARG SGX_MODE

# config/build params
ARG FABRIC_REPO=https://github.com/hyperledger/fabric.git
ARG FABRIC_VERSION=2.2.0
ARG FPC_REPO_URL=https://github.com/hyperledger-labs/fabric-private-chaincode.git
ARG FPC_REPO_BRANCH_TAG_OR_COMMIT=master
ARG GIT_USER_NAME=tester
ARG GIT_USER_EMAIL=tester@fpc


# make sure we have a valid git user (needed for the git am when patching fabric)
RUN git config --global user.name "${GIT_USER_NAME}" \
 && git config --global user.email "${GIT_USER_EMAIL}"


# Get Fabric
ENV FABRIC_PATH=${GOPATH}/${FABRIC_REL_PATH}
RUN git clone --branch v${FABRIC_VERSION} ${FABRIC_REPO} ${FABRIC_PATH}
# Note: could add --single-branch to below to speed-up and keep size smaller. But for now for a dev-image better keep complete repo

# Get FPC
ENV FPC_PATH=${GOPATH}/${FPC_REL_PATH}
# We copy context so we can use that to potentially get local .git as repo ...
COPY .git /tmp/cloned-local-fpc-git-repo/
RUN git \
       -c submodule.interpreters/wasm-micro-runtime.update=none -c submodule.ccf_transaction_processor/CCF.update=none \
	clone --recurse ${FPC_REPO_URL} ${FPC_PATH} \
  && cd ${FPC_PATH} \
  && git checkout --recurse ${FPC_REPO_BRANCH_TAG_OR_COMMIT}
# Notes:
# - the -c submodule's are to prevent dragging in large but unneeded sub-sub-modules of pdo ...


# peer builder container (Ephemeral)
#------------------------------------
FROM common as peer-builder

# import global vars
ARG SGX_MODE
ENV SGX_MODE=${SGX_MODE}
ARG FPC_ATTESTATION_TYPE
ENV FPC_ATTESTATION_TYPE=${FPC_ATTESTATION_TYPE}

# Build FPC peer including plugins and tlcc support
RUN cd ${FPC_PATH} \
 && make fpc-peer \
 && make fpc-peer-cli


# peer Container
#------------------------
# Note we don't need all the build support for that,
# so just start from base and copy the necessary built binaries/scripts
FROM hyperledger/fabric-private-chaincode-base-rt:${FPC_VERSION} as peer

# import global vars
ARG FABRIC_REL_PATH
ARG FPC_REL_PATH
ARG SGX_MODE
ARG FPC_ATTESTATION_TYPE

# local vars
# note these envs are _not_ inhereted from above as we start from base-rt !
ARG FABRIC_PATH=${GOPATH}/${FABRIC_REL_PATH}
ARG FPC_PATH=${GOPATH}/${FPC_REL_PATH}
ARG FABRIC_BIN_DIR=${FABRIC_PATH}/build/bin
ARG FPC_CMDS=${FPC_PATH}/fabric/bin

ENV FABRIC_PATH=${FABRIC_PATH}
ENV FABRIC_BIN_DIR=${FABRIC_BIN_DIR}
ENV FPC_PATH=${FPC_PATH}
ENV FPC_CMDS=${FPC_CMDS}
ENV SGX_MODE=${SGX_MODE}
ENV FPC_ATTESTATION_TYPE=${FPC_ATTESTATION_TYPE}

RUN apt-get update \
  && apt-get install -y -q \
    docker.io \
    jq \
    ${APT_ADD_PKGS}


# components we need
# - peer cli wrapper
COPY --from=peer-builder ${FPC_CMDS} ${FPC_CMDS}
# - fabric binariers
COPY --from=peer-builder ${FABRIC_BIN_DIR} ${FABRIC_BIN_DIR}
# - (sgx) config
COPY --from=peer-builder ${FPC_PATH}/config ${FPC_PATH}/config
# - tlcc ( libtl.so & enclave.so)
#   Note we need the (symlinked) path through tlcc rather than through tlcc_enclave
COPY --from=peer-builder ${FPC_PATH}/tlcc/enclave/lib ${FPC_PATH}/tlcc/enclave/lib
# - ercc binary
COPY --from=peer-builder ${FPC_PATH}/ercc/ercc ${FPC_PATH}/ercc/ercc
# - external builders itself ..
COPY --from=peer-builder ${FPC_PATH}/fabric/externalBuilder ${FPC_PATH}/fabric/externalBuilder
# - and for host-based run also ecc and libs
COPY --from=peer-builder ${FPC_PATH}/ecc/ecc ${FPC_PATH}/ecc/ecc
COPY --from=peer-builder ${FPC_PATH}/ecc_enclave/_build/lib/libsgxcc.so ${FPC_PATH}/ecc_enclave/_build/lib/libsgxcc.so

CMD [${FPC_CMDS}/peer.sh node start]


# cc-builder Container
#------------------------
FROM common as cc-builder

# import global vars
ARG SGX_MODE
ENV SGX_MODE=${SGX_MODE}
# Note: Only the untrusted part of the SDK depends on FPC_ATTESTATION_TYPE; the trusted parts
#  compiled into `enclave.so`, the output of a running cc-builder, do not. So it does not
#  matter which way the SDK is built regarding attestation and we do not have to "specialize"
#  for FPC_ATTESTATION_TYPE as we do for `peer`.  But as properties of `enclave.so` depend
#  on SGX_MODE, we do have to specialize for that ...

WORKDIR ${FPC_PATH}

RUN make fpc-sdk



# Dev Container
#------------------------
FROM common as dev

# import global vars
ARG FPC_REL_PATH
ARG SGX_MODE

ENV SGX_MODE=${SGX_MODE}

RUN apt-get install -y -q \
	psmisc \
	bc \
	docker-compose\
	${APT_DEV_ADD_PKGS}

WORKDIR ${FPC_PATH}
