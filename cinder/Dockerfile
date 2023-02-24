# Cinder driver container for Dell EMC VNX and Unity
# Using openstack-cinder-volume base image

FROM registry.redhat.io/rhosp-rhel8/openstack-cinder-volume:16.2.4

ARG storops_version=1.2.11

LABEL maintainer="Dell EMC" \
      description="Red Hat OpenStack Platform 16.2 cinder-volume Dell EMC VNX and Unity" \
      summary="Red Hat OpenStack Platform 16.2 cinder-volume Dell EMC VNX and Unity" \
      name="dellemc/openstack-cinder-volume-dellemc-rhosp16" \
      vendor="Dell EMC" \
      version="4.4.1" \
      storops=${storops_version} \
      naviseccli="7.33.9.1.55"

# Switch to root to install packages
USER root

# Install storops
RUN pip3 install https://github.com/emc-openstack/storops/archive/v${storops_version}.tar.gz

# libnsl needed by naviseccli has no longer provided by base image, need to install it
ARG libnsl=libnsl-2.28-151.el8.x86_64.rpm
RUN curl -sSOL https://github.com/emc-openstack/naviseccli/raw/master/${libnsl}
RUN yum localinstall -y ${libnsl}

# Install naviseccli
ARG naviseccli=NaviCLI-Linux-64-x86-en_US-7.33.9.1.55-1.x86_64.rpm
RUN curl -sSOL https://github.com/emc-openstack/naviseccli/raw/master/${naviseccli}
RUN yum localinstall -y ${naviseccli}

# Add license info
RUN curl -OL https://raw.githubusercontent.com/emc-openstack/osp-deploy/master/LICENSE.txt
RUN mkdir /licenses
RUN cp LICENSE.txt /licenses/licensing.txt

# Switch to cinder user
USER cinder
