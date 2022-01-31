
FROM quay.io/cybozu/ubuntu:20.04
MAINTAINER Gerrit Botha "gerrit.botha@uct.ac.za"

RUN apt-get -qq update && apt-get -qq -y install git pip autoconf libbz2-dev liblzma-dev ncurses-dev \
    libsodium-dev libcurl4-gnutls-dev \
    ca-certificates pkg-config \
    gcc make automake autoconf libtool \
    bzip2 zlib1g-dev libssl-dev libedit-dev \
    ninja-build cmake udev libc6-dev libglib2.0-dev \
    && apt-get -qq -y autoremove \
    && apt-get autoclean \
    && rm -rf /var/lib/apt/lists/{apt,dpkg,cache,log} /tmp/* /var/tmp/*

# Install C implementation of crypt4gh
WORKDIR /opt

RUN git clone https://github.com/samtools/htslib.git
RUN git clone https://github.com/samtools/samtools.git
RUN git clone https://github.com/samtools/htslib-crypt4gh.git

RUN cd htslib \
    && git submodule update --init --recursive \
    && autoreconf -i \
    && ./configure --prefix=/opt/samtools --enable-plugins --enable-libcurl  \
    && make \
    && make install

WORKDIR /opt
RUN cd samtools \
    && autoreconf -i \
    && ./configure --prefix=/opt/samtools --with-htslib=/opt/samtools LDFLAGS="-Wl,-R$/opt/samtools/lib" \
    && make \
    && make install

WORKDIR /opt
RUN cd htslib-crypt4gh \
    && autoreconf -i \
    && ./configure --prefix=/opt/samtools \
    && make \
    && make install

ENV LD_LIBRARY_PATH $LD_LIBRARY_PATH:/opt/samtools/lib

# Install Python implementation of crypt4gh
RUN pip install crypt4gh

# Install crypt4gfs
RUN pip install -U pip
RUN pip install meson pytest

WORKDIR /opt
RUN git clone https://github.com/libfuse/libfuse.git \
    && cd libfuse \
    && git checkout fuse-3.10.0 \
    && mkdir build \
    && cd build \
    && meson .. \
    && ninja \
    && ninja install

RUN git clone https://github.com/libfuse/sshfs.git \
    && cd sshfs \
    && git checkout sshfs-3.7.0 \
    && mkdir build \
    && cd build \
    && meson .. \
    && ninja \
    && ninja install

#RUN pip install crypt4ghfs
RUN pip install crypt4ghfs

# Switch back to the ubuntu user so this tool (and the files written) are not owned by root
RUN groupadd -r -g 1000 ubuntu && useradd -r -g ubuntu -u 1000 -m ubuntu
USER ubuntu

WORKDIR /home/ubuntu

ENV PATH /opt/samtools/bin:$PATH
ENV LD_LIBRARY_PATH LD_LIBRARY_PATH:/usr/local/lib/x86_64-linux-gnu/
ENV LD_LIBRARY_PATH $LD_LIBRARY_PATH:/opt/samtools/lib
