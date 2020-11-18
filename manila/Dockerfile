# Manila driver container for Dell EMC Unity
# Using openstack-manila-share base image

FROM registry.redhat.io/rhosp-rhel8/openstack-manila-share:16.1

ARG storops_version=1.2.7

LABEL maintainer="Dell EMC" \
      description="Red Hat OpenStack Platform 16.1 manila-share Dell EMC Unity" \
      summary="Red Hat OpenStack Platform 16.1 manila-share Dell EMC Unity" \
      name="dellemc/openstack-manila-share-dellemc-rhosp16" \
      vendor="Dell EMC" \
      version="4.2.1" \
      storops=${storops_version}

# Switch to root to install packages
USER root

# Install storops
RUN pip3 install https://github.com/emc-openstack/storops/archive/v${storops_version}.tar.gz

# Add license info
RUN curl -OL https://raw.githubusercontent.com/emc-openstack/osp-deploy/master/LICENSE.txt
RUN mkdir /licenses
RUN cp LICENSE.txt /licenses/licensing.txt

# Switch to manila user
USER manila
