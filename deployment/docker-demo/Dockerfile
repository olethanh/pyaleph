# Monolithic Docker image for easy setup of an Aleph.im node in demo scenarios.

FROM ubuntu:20.04

ENV DEBIAN_FRONTEND noninteractive

# Install Python dependencies
RUN apt-get update && apt-get -y upgrade && apt-get install -y \
     python3 \
     python3-dev \
     python3-pip \
     python3-venv \
     build-essential \
     git

# Install system dependencies
RUN apt-get install -y \
     libsnappy-dev \
     zlib1g-dev \
     libbz2-dev \
     libgflags-dev \
     liblz4-dev \
     librocksdb-dev \
     libgmp-dev \
     libsecp256k1-dev \
     pkg-config \
     libssl-dev \
     libleveldb-dev \
     libyaml-dev


# === Install MongoDB, IPFS and Supervisord ===

# Install MongoDB
RUN apt-get install -y mongodb
RUN mkdir /run/mongodb
RUN chown mongodb:mongodb /run/mongodb

# Install IPFS
RUN apt-get install -y wget
RUN wget https://ipfs.io/ipns/dist.ipfs.io/go-ipfs/v0.7.0/go-ipfs_v0.7.0_linux-amd64.tar.gz
RUN tar -xvzf go-ipfs_v0.7.0_linux-amd64.tar.gz -C /opt/
RUN ln -s /opt/go-ipfs/ipfs /usr/local/bin/
RUN mkdir /var/lib/ipfs
# - User 'ipfs' to run IPFS -
RUN useradd -s /bin/bash ipfs
RUN chown -R ipfs:ipfs /var/lib/ipfs

# Install Supervisord
RUN apt-get install -y supervisor


# ===  Create an unprivileged user to install Python dependencies ===

# - User 'source' to install code and dependencies -
RUN useradd -s /bin/bash source
RUN mkdir /opt/venv
RUN chown source:source /opt/venv

# - User 'aleph' to run the code itself
RUN useradd -s /bin/bash aleph
RUN mkdir /opt/pyaleph
RUN chown aleph:aleph /opt/pyaleph
USER aleph
# ENV is user-specific
ENV IPFS_PATH /var/lib/ipfs


# === Install Python environment and dependencies ===
USER source

# Create virtualenv
RUN python3 -m venv /opt/venv

# Install pip
ENV PIP_NO_CACHE_DIR yes
RUN /opt/venv/bin/python3 -m pip install --upgrade pip
RUN /opt/venv/bin/python3 -m pip install --upgrade wheel
ENV PATH="/opt/venv/bin:${PATH}"

# === Copy source code ===
COPY setup.py /opt/pyaleph/
COPY setup.cfg /opt/pyaleph/
COPY src /opt/pyaleph/src

COPY tests /opt/pyaleph/tests

COPY .git /opt/pyaleph/.git


# === Install the application and dependencies ===

# Setup directories for `python setup.py develop`
USER root
RUN mkdir /opt/pyaleph/src/pyaleph.egg-info
RUN mkdir /opt/pyaleph/.eggs
RUN chown -R source:source /opt/pyaleph/src /opt/pyaleph/.eggs

# Install PyAleph source
USER source
WORKDIR /opt/pyaleph
RUN python setup.py develop
RUN /opt/venv/bin/pip install --use-feature=2020-resolver -e ".[nuls2,testing]"

# Fix an issue with Aiohttp not working
RUN /opt/venv/bin/pip install aiohttp==3.7.2

# === Run PyAleph,MongoDB and IPFS using supervisord ===
USER root
COPY deployment/docker-demo/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
# Entrypoint script used to initialize IPFS and launch supervisord
COPY deployment/docker-demo/entrypoint.sh /entrypoint.sh
CMD ["/entrypoint.sh"]

# IPFS Swarm
EXPOSE 4001
# IPFS WebUI
EXPOSE 5001
# IPFS Gateway
EXPOSE 8080

# PyAleph API
EXPOSE 8000
# PyAleph p2p network
EXPOSE 4024 4025

VOLUME /var/lib/mongodb
VOLUME /var/lib/ipfs
