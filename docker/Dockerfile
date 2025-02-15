# ==================== Building Model Layer ===========================
# This is a little trick to improve caching and minimize rebuild time
# and bandwidth. Note that RUN commands only cache-miss if the prior layers
# miss, or the dockerfile changes prior to this step.
# To update these patch files, be sure to run build with --no-cache
FROM alpine as model_data
RUN apk --no-cache --update-cache add wget
WORKDIR /data/patch_experts

RUN wget -q https://www.dropbox.com/s/7na5qsjzz8yfoer/cen_patches_0.25_of.dat &&\
    wget -q https://www.dropbox.com/s/k7bj804cyiu474t/cen_patches_0.35_of.dat &&\
    wget -q https://www.dropbox.com/s/ixt4vkbmxgab1iu/cen_patches_0.50_of.dat &&\
    wget -q https://www.dropbox.com/s/2t5t1sdpshzfhpj/cen_patches_1.00_of.dat

## ==================== Install Ubuntu Base libs ===========================
## This will be our base image for OpenFace, and also the base for the compiler
## image. We only need packages which are linked

FROM ubuntu:18.04 as ubuntu_base

LABEL maintainer="Michael McDermott <mikemcdermott23@gmail.com>"

ARG DEBIAN_FRONTEND=noninteractive

# todo: minimize this even more
RUN apt-get update -qq &&\
    apt-get install -qq curl &&\
    apt-get install -qq wget &&\
    apt-get install -qq --no-install-recommends \
        libopenblas-dev liblapack-dev \
        libavcodec-dev libavformat-dev libswscale-dev \
        libtbb2 libtbb-dev libjpeg-dev \
        libpng-dev libtiff-dev &&\
    rm -rf /var/lib/apt/lists/*

## ==================== Build-time dependency libs ======================
## This will build and install opencv and dlib into an additional dummy
## directory, /root/diff, so we can later copy in these artifacts,
## minimizing docker layer size
## Protip: ninja is faster than `make -j` and less likely to lock up system
FROM ubuntu_base as cv_deps

WORKDIR /root/build-dep
ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update -qq && apt-get install -qq -y \
        cmake ninja-build pkg-config build-essential checkinstall\
        g++-8 &&\
    rm -rf /var/lib/apt/lists/* &&\
    update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-8 800 --slave /usr/bin/g++ g++ /usr/bin/g++-8

##        llvm clang-3.7 libc++-dev libc++abi-dev  \
## ==================== Building dlib ===========================

RUN curl http://dlib.net/files/dlib-19.13.tar.bz2 -LO &&\
    tar xf dlib-19.13.tar.bz2 && \
    rm dlib-19.13.tar.bz2 &&\
    mv dlib-19.13 dlib &&\
    mkdir -p dlib/build &&\
    cd dlib/build &&\
    cmake -DCMAKE_BUILD_TYPE=Release -G Ninja .. &&\
    ninja && \
    ninja install && \
    DESTDIR=/root/diff ninja install &&\
    ldconfig

## ==================== Building OpenCV ======================
ENV OPENCV_VERSION=4.1.0

RUN curl https://github.com/opencv/opencv/archive/${OPENCV_VERSION}.tar.gz -LO &&\
    tar xf ${OPENCV_VERSION}.tar.gz && \
    rm ${OPENCV_VERSION}.tar.gz &&\
    mv opencv-${OPENCV_VERSION} opencv && \
    mkdir -p opencv/build && \
    cd opencv/build && \
    cmake -D CMAKE_BUILD_TYPE=RELEASE \
        -D CMAKE_INSTALL_PREFIX=/usr/local \
        -D WITH_TBB=ON -D WITH_CUDA=OFF \
        -DWITH_QT=OFF -DWITH_GTK=OFF\
        -G Ninja .. && \
    ninja && \
    ninja install &&\
    DESTDIR=/root/diff ninja install

## ==================== Building OpenFace ===========================
FROM cv_deps as openface
WORKDIR /root/openface

COPY ./OpenFace/ ./

COPY --from=model_data /data/patch_experts/* \
    /root/openface/lib/local/LandmarkDetector/model/patch_experts/

RUN mkdir -p build && cd build && \
    cmake -D CMAKE_BUILD_TYPE=RELEASE -G Ninja .. && \
    ninja &&\
    DESTDIR=/root/diff ninja install


## ==================== Streamline container ===========================
## Clean up - start fresh and only copy in necessary stuff
## This shrinks the image from ~8 GB to ~1.6 GB
FROM ubuntu_base as final

WORKDIR /root

# Copy in only necessary libraries
COPY --from=openface /root/diff /

# Since we "imported" the build artifacts, we need to reconfigure ld
RUN ldconfig

## ==================== Flamingo Set Up ===========================

# Setting up non-interactive env stuff
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

ENV CONDA_ROOT /opt/miniconda
ENV CONDA_ENV_NAME flamingo-fer
ENV PATH ${CONDA_ROOT}/envs/${CONDA_ENV_NAME}/bin:${CONDA_ROOT}/bin:${PATH}

RUN wget -q http://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh
RUN bash Miniconda3-latest-Linux-x86_64.sh -p $CONDA_ROOT -b
RUN rm Miniconda3-latest-Linux-x86_64.sh
RUN conda update -y conda

ADD configs/environment.yml /tmp
RUN conda env create -f /tmp/environment.yml -n $CONDA_ENV_NAME python=3.8
RUN echo "source activate ${CONDA_ENV_NAME}" > ~/.bashrc

SHELL ["conda", "run", "--no-capture-output", "-n", "flamingo-fer", "/bin/bash", "-c"]

# ML Related preloading
RUN python -c "import nltk; nltk.download(\"punkt\")"
RUN python -c "import nltk; nltk.download(\"stopwords\")"
RUN python -c "from transformers import AutoModelForSequenceClassification; \
               AutoModelForSequenceClassification.from_pretrained(\"facebook/bart-large-mnli\")"
RUN python -c "from transformers import AutoTokenizer; \
               AutoTokenizer.from_pretrained(\"facebook/bart-large-mnli\")"
