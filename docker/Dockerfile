ARG CUDA_VERSION=cu12

FROM nvidia/cuda:12.4.1-devel-ubuntu22.04 AS cu12
ENV CUDA_VERSION_SHORT=cu121

FROM nvidia/cuda:11.8.0-devel-ubuntu22.04 AS cu11
ENV CUDA_VERSION_SHORT=cu118

FROM ${CUDA_VERSION} AS final

ARG PYTHON_VERSION=3.10

RUN apt-get update -y && apt-get install -y software-properties-common wget vim git curl openssh-server ssh sudo &&\
    apt-get install libibverbs1 ibverbs-providers ibverbs-utils librdmacm1 libibverbs-dev rdma-core -y &&\
    curl https://sh.rustup.rs -sSf | sh -s -- -y &&\
    add-apt-repository ppa:deadsnakes/ppa -y && apt-get update -y && apt-get install -y --no-install-recommends \
    ninja-build rapidjson-dev libgoogle-glog-dev gdb python${PYTHON_VERSION} python${PYTHON_VERSION}-dev python${PYTHON_VERSION}-venv \
    && apt-get clean -y && rm -rf /var/lib/apt/lists/* && cd /opt && python3 -m venv py3

ENV PATH=/opt/py3/bin:$PATH

# install nccl manually for cuda11.8
RUN if [ "$CUDA_VERSION_SHORT" = "cu118" ]; then \
    git clone --depth=1 --branch v2.22.3-1 https://github.com/NVIDIA/nccl.git &&\
    cd nccl && make -j$(nproc) src.build &&\
    mv build/include/* /usr/local/include &&\
    mkdir -p /usr/local/nccl/lib &&\
    mv build/lib/lib* /usr/local/nccl/lib/ &&\
    cd .. && rm -rf nccl ; \
    fi


ENV LD_LIBRARY_PATH=/usr/local/nccl/lib:$LD_LIBRARY_PATH


RUN --mount=type=cache,target=/root/.cache/pip python3 -m pip install --upgrade pip setuptools==69.5.1 &&\
    python3 -m pip install cmake packaging wheel

ENV NCCL_LAUNCH_MODE=GROUP

# Should be in the lmdeploy root directory when building docker image
COPY . /opt/lmdeploy

WORKDIR /opt/lmdeploy

RUN --mount=type=cache,target=/root/.cache/pip cd /opt/lmdeploy &&\
    python3 -m pip install -r requirements_cuda.txt --extra-index-url https://download.pytorch.org/whl/${CUDA_VERSION_SHORT} &&\
    mkdir -p build && cd build &&\
    sh ../generate.sh &&\
    ninja -j$(nproc) && ninja install &&\
    cd .. &&\
    python3 -m pip install -e . &&\
    rm -rf build

# use locally built nccl for cuda11.8
RUN if [ "$CUDA_VERSION_SHORT" = "cu118" ]; then python3 -m pip uninstall -y nvidia-nccl-cu11 ; fi
# torch 2.7 brings in nvidia-nccl-cu12 2.26.x. Somehow, it causes performance degradation of pytorch engine
RUN if [ "$CUDA_VERSION_SHORT" = "cu121" ]; then python3 -m pip install nvidia-nccl-cu12==2.25.1; fi

ENV LD_LIBRARY_PATH=/opt/lmdeploy/install/lib:$LD_LIBRARY_PATH
ENV PATH=/opt/lmdeploy/install/bin:$PATH

# explicitly set ptxas path for triton
ENV TRITON_PTXAS_PATH=/usr/local/cuda/bin/ptxas
