# Cinder driver container for Dell Unity
# Using openstack-cinder-volume base image

FROM registry.redhat.io/rhosp-rhel9/openstack-cinder-volume:17.1

ARG storops_version=1.2.11

LABEL maintainer="Dell" \
      description="Red Hat OpenStack Platform 17.1 cinder-volume Dell Unity" \
      summary="Red Hat OpenStack Platform 17.1 cinder-volume Dell Unity" \
      name="dellemc/openstack-cinder-volume-dellemc-rhosp17" \
      vendor="Dell" \
      version="5.0.0" \
      storops=${storops_version}

# Switch to root to install packages
USER root

# Install storops
RUN python3 -m ensurepip
RUN pip3 install https://github.com/emc-openstack/storops/archive/v${storops_version}.tar.gz

# Add license info
RUN curl -OL https://raw.githubusercontent.com/emc-openstack/osp-deploy/master/LICENSE.txt
RUN mkdir /licenses
RUN cp LICENSE.txt /licenses/licensing.txt

# Switch to cinder user
USER cinder
