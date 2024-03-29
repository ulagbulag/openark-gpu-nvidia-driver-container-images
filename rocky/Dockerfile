ARG CUDA_VERSION=12.0.0
ARG ROCKYLINUX_VERSION=9.1

# Build CUDA Image
FROM quay.io/rockylinux/rockylinux:${ROCKYLINUX_VERSION} as base

ARG CUDA_VERSION
ARG DRIVER_VERSION
ARG ROCKYLINUX_VERSION
ENV CUDA_VERSION=${CUDA_VERSION}
ENV DRIVER_VERSION=${DRIVER_VERSION}
ENV ROCKYLINUX_VERSION=${ROCKYLINUX_VERSION}

LABEL maintainer "Ho Kim <ho.kim@ulagbulag.io>"

RUN NVARCH=$(uname -m | sed 's/^aarch64$/arm64/g' | sed 's/^arm64$/sbsa/g') \
    && echo -e "\
    [cuda]\n \
    name=cuda\n \
    baseurl=https://developer.download.nvidia.com/compute/cuda/repos/rhel${ROCKYLINUX_VERSION/\.[0-9\.]*}/${NVARCH}\n \
    enabled=1\n \
    gpgcheck=1\n \
    gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-NVIDIA\n \
    " | sed 's/^ *\(.*\)$/\1/g' > /etc/yum.repos.d/cuda.repo \
    && NVIDIA_GPGKEY_SUM=d0664fbbdb8c32356d45de36c5984617217b2d0bef41b93ccecd326ba3b80c87 \
    && curl -fsSL https://developer.download.nvidia.com/compute/cuda/repos/rhel${ROCKYLINUX_VERSION/\.[0-9\.]*}/${NVARCH}/D42D0685.pub \
    | sed '/^Version/d' > /etc/pki/rpm-gpg/RPM-GPG-KEY-NVIDIA \
    && echo "$NVIDIA_GPGKEY_SUM  /etc/pki/rpm-gpg/RPM-GPG-KEY-NVIDIA" | sha256sum -c --strict - \
    # For libraries in the cuda-compat-* package: https://docs.nvidia.com/cuda/eula/index.html#attachment-a
    && CUDA_VERSION_WITHOUT_PATCH="$(echo "${CUDA_VERSION%\.[0-9]*}" | sed 's/\./-/g')" \
    && dnf upgrade -y && dnf install -y \
    ca-certificates \
    cuda-compat-${CUDA_VERSION_WITHOUT_PATCH}-${DRIVER_VERSION} \
    cuda-cudart-${CUDA_VERSION_WITHOUT_PATCH} \
    curl \
    dnf-utils \
    gcc \
    glibc.i686 \
    kmod \
    make \
    util-linux \
    # Remove cuda repository to avoid GPG errors
    && rm -f /etc/yum.repos.d/cuda.repo \
    && yum clean all \
    && rm -rf /var/cache/yum/* \
    # nvidia-docker 1.0
    && echo "/usr/local/nvidia/lib" >> /etc/ld.so.conf.d/nvidia.conf \
    && echo "/usr/local/nvidia/lib64" >> /etc/ld.so.conf.d/nvidia.conf

ENV PATH=/usr/local/nvidia/bin:/usr/local/cuda/bin:${PATH}
ENV LD_LIBRARY_PATH=/usr/local/nvidia/lib:/usr/local/nvidia/lib64

# nvidia-container-runtime
ENV NVIDIA_VISIBLE_DEVICES=void
ENV NVIDIA_DRIVER_CAPABILITIES=compute,utility

# Build NVIDIA Driver Image

RUN curl -fsSL -o /usr/local/bin/donkey https://github.com/3XX0/donkey/releases/download/v1.1.0/donkey \
    && curl -fsSL -o /usr/local/bin/extract-vmlinux https://raw.githubusercontent.com/torvalds/linux/master/scripts/extract-vmlinux \
    && chmod +x /usr/local/bin/donkey /usr/local/bin/extract-vmlinux

#ARG BASE_URL=http://us.download.nvidia.com/XFree86/Linux-x86_64
ARG BASE_URL=https://us.download.nvidia.com/tesla

# Install the userspace components and copy the kernel module sources.
RUN cd /tmp && \
    curl -fSsl -O ${BASE_URL}/${DRIVER_VERSION}/NVIDIA-Linux-x86_64-${DRIVER_VERSION}.run && \
    sh NVIDIA-Linux-x86_64-${DRIVER_VERSION}.run -x && \
    cd NVIDIA-Linux-x86_64-${DRIVER_VERSION} && \
    ./nvidia-installer --silent \
    --no-kernel-module \
    --install-compat32-libs \
    --no-nouveau-check \
    --no-nvidia-modprobe \
    --no-rpms \
    --no-backup \
    --no-check-for-alternate-installs \
    --no-libglx-indirect \
    --no-install-libglvnd \
    --x-prefix=/tmp/null \
    --x-module-path=/tmp/null \
    --x-library-path=/tmp/null \
    --x-sysconfig-path=/tmp/null && \
    mkdir -p /usr/src/nvidia-${DRIVER_VERSION} && \
    mv LICENSE mkprecompiled kernel /usr/src/nvidia-${DRIVER_VERSION} && \
    sed '9,${/^\(kernel\|LICENSE\)/!d}' .manifest > /usr/src/nvidia-${DRIVER_VERSION}/.manifest && \
    rm -rf /tmp/*

COPY nvidia-driver /usr/local/bin

WORKDIR /usr/src/nvidia-${DRIVER_VERSION}

ENTRYPOINT ["nvidia-driver", "init"]
