# Modified from the softlearning Docker file
# https://github.com/rail-berkeley/softlearning

ARG UBUNTU_VERSION=16.04
ARG ARCH=
ARG CUDA=10.0

FROM nvidia/cudagl${ARCH:+-$ARCH}:${CUDA}-base-ubuntu${UBUNTU_VERSION} as base
# ARCH and CUDA are specified again because the FROM directive resets ARGs
# (but their default value is retained if set previously)

ARG UBUNTU_VERSION
ARG ARCH
ARG CUDA
ARG CUDNN=7.6.5.32-1

SHELL ["/bin/bash", "-c"]


ENV DEBIAN_FRONTEND="noninteractive"
# See http://bugs.python.org/issue19846
ENV LANG=C.UTF-8 LC_ALL=C.UTF-8
ENV PATH /opt/conda/bin:$PATH

# don't know what this does??
RUN apt-get update --fix-missing && apt-get install -y wget bzip2 ca-certificates \
    libglib2.0-0 libxext6 libsm6 libxrender1 \
    git mercurial subversion

# install anaconda
RUN wget --quiet https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O /tmp/miniconda.sh && \
    /bin/bash /tmp/miniconda.sh -b -p /opt/conda && \
    rm /tmp/miniconda.sh && \
    ln -s /opt/conda/etc/profile.d/conda.sh /etc/profile.d/conda.sh && \
    echo ". /opt/conda/etc/profile.d/conda.sh" >> /etc/bash.bashrc

# don't know what this does either
RUN apt-get install -y curl grep sed dpkg && \
    TINI_VERSION=`curl https://github.com/krallin/tini/releases/latest | grep -o "/v.*\"" | sed 's:^..\(.*\).$:\1:'` && \
    curl -L "https://github.com/krallin/tini/releases/download/v${TINI_VERSION}/tini_${TINI_VERSION}.deb" > tini.deb && \
    dpkg -i tini.deb && \
    rm tini.deb && \
    apt-get clean \
    && rm -rf /var/lib/apt/lists/*


RUN conda update -y --name base conda \
    && conda clean --all -y


# ========== Tensorflow dependencies ==========
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        cuda-command-line-tools-${CUDA/./-} \
        cuda-cublas-${CUDA/./-} \
        cuda-cufft-${CUDA/./-} \
        cuda-curand-${CUDA/./-} \
        cuda-cusolver-${CUDA/./-} \
        cuda-cusparse-${CUDA/./-} \
        curl \
        libcudnn7=${CUDNN}+cuda${CUDA} \
        libfreetype6-dev \
        libhdf5-serial-dev \
        libzmq3-dev \
        pkg-config \
        software-properties-common \
        zip \
        unzip \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN [ ${ARCH} = ppc64le ] || (apt-get update && \
        apt-get install nvinfer-runtime-trt-repo-ubuntu${UBUNTU_VERSION/./}-5.0.2-ga-cuda${CUDA} \
        && apt-get update \
        && apt-get install -y --no-install-recommends libnvinfer5=5.0.2-1+cuda${CUDA} \
        && apt-get clean \
        && rm -rf /var/lib/apt/lists/*)

# For CUDA profiling, TensorFlow requires CUPTI.
ENV LD_LIBRARY_PATH /usr/local/cuda/extras/CUPTI/lib64:$LD_LIBRARY_PATH

# ========== Repo dependencies ==========
RUN apt-get update -y \
    && apt-get install -y --no-install-recommends \
        build-essential \
        curl \
        git \
        gnupg2 \
        make \
        cmake \
        ffmpeg \
        swig \
        libz-dev \
        unzip \
        zlib1g-dev \
        libglfw3 \
        libglfw3-dev \
        libxrandr2 \
        libxinerama-dev \
        libxi6 \
        libxcursor-dev \
        libgl1-mesa-dev \
        libgl1-mesa-glx \
        libglew-dev \
        libosmesa6-dev \
        lsb-release \
        ack-grep \
        patchelf \
        vim \
        wget \
        xpra \
        xserver-xorg-dev \
        xvfb \
    && export CLOUD_SDK_REPO="cloud-sdk-$(lsb_release -c -s)" \
    && echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" \
            | tee -a /etc/apt/sources.list.d/google-cloud-sdk.list \
    && curl https://packages.cloud.google.com/apt/doc/apt-key.gpg \
            | apt-key add - \
    && apt-get update -y \
    && apt-get install -y google-cloud-sdk \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*


# ========= MuJoCo ===============
COPY ./install_mujoco.py /tmp/

RUN /tmp/install_mujoco.py --mujoco-path=/root/.mujoco --versions 1.50 2.00 \
    && ln -s /root/.mujoco/mujoco200_linux /root/.mujoco/mujoco200 \
    && rm /tmp/install_mujoco.py

ENV LD_LIBRARY_PATH /root/.mujoco/mjpro150/bin:${LD_LIBRARY_PATH}
ENV LD_LIBRARY_PATH /root/.mujoco/mujoco200/bin:${LD_LIBRARY_PATH}
ENV LD_LIBRARY_PATH /root/.mujoco/mujoco200_linux/bin:${LD_LIBRARY_PATH}

# This is a hack required to make mujocopy to compile in gpu mode
RUN mkdir -p /usr/lib/nvidia-000
ENV LD_LIBRARY_PATH ${LD_LIBRARY_PATH}:/usr/lib/nvidia-000


# ========== Conda Environment ==========
COPY ./environment.yml /tmp/environment.yml

# NOTE: Don't separate the Mujoco key echo and remove commands into separate
# run commands! Otherwise your key will be readable by anyone who has access
# To the container. We need the key in order to compile mujoco_py.
# TODO this is not good but it can't find the key otherwise
#RUN echo "${MJKEY}" > ./mjkey.txt 
COPY ./mjkey.txt /root/.mujoco/
RUN conda env update -f /tmp/environment.yml \
    && conda clean --all -y 

RUN mkdir /root/meld
WORKDIR /root/meld

# hoping this fixes opengl error
# (TODO) figure out if this does anything
ENV DISPLAY :0

# activate conda env
RUN echo "source activate meld" >> /root/.bashrc \
    && echo "cd /root/meld" >> /root/.bashrc
ENV PATH /opt/conda/envs/meld/bin:$PATH

# add code to python path
ENV PYTHONPATH=".:$PYTHONPATH"

# manually install custom tf-agents
ADD ../agents-0.3.0 /root/agents-0.3.0
RUN pip install -e /root/agents-0.3.0

