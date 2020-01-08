# Cinder driver container for Dell EMC VNX and Unity
# Using openstack-cinder-volume base image

FROM registry.redhat.io/rhosp15-rhel8/openstack-cinder-volume

ARG storops_version=1.2.2

LABEL maintainer="Dell EMC" \
      description="Red Hat OpenStack Platform 15.0 cinder-volume Dell EMC VNX and Unity" \
      summary="Red Hat OpenStack Platform 15.0 cinder-volume Dell EMC VNX and Unity" \
      name="dellemc/openstack-cinder-volume-dellemc-rhosp15" \
      vendor="Dell EMC" \
      version="3.0.0" \
      storops=${storops_version} \
      naviseccli="7.33.9.1.55"

# Switch to root to install packages
USER root

# Keep the version of packages installed via rpm in base image.
# Then installing storops won't upgrade existing packages.
RUN pip3 freeze > packages_via_rpm.txt

# Install storops
RUN pip3 install --constraint packages_via_rpm.txt \
    https://github.com/emc-openstack/storops/archive/r${storops_version}.tar.gz

# Verify storops has compatible dependencies.
# Image build will fail if storops misses any dependency.
RUN ! pip3 check | grep storops

# Install naviseccli
ARG naviseccli=NaviCLI-Linux-64-x86-en_US-7.33.9.1.55-1.x86_64.rpm
RUN curl -sSOL https://github.com/emc-openstack/naviseccli/raw/master/${naviseccli}
RUN yum localinstall -y ${naviseccli}
RUN rm -rf ${naviseccli}

# Add license info
RUN curl -OL https://raw.githubusercontent.com/emc-openstack/osp-deploy/master/LICENSE.txt
RUN mkdir /licenses
RUN cp LICENSE.txt /licenses/licensing.txt

# Switch to cinder user
USER cinder
