# Manila driver container for Dell EMC Unity
# Using openstack-manila-share base image

FROM registry.redhat.io/rhosp15-rhel8/openstack-manila-share

ARG storops_version=1.2.2

LABEL maintainer="Dell EMC" \
      description="Red Hat OpenStack Platform 15.0 manila-share Dell EMC Unity" \
      summary="Red Hat OpenStack Platform 15.0 manila-share Dell EMC Unity" \
      name="dellemc/openstack-manila-share-dellemc-rhosp15" \
      vendor="Dell EMC" \
      version="3.0.0" \
      storops=${storops_version}

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

# Add license info
RUN curl -OL https://raw.githubusercontent.com/emc-openstack/osp-deploy/master/LICENSE.txt
RUN mkdir /licenses
RUN cp LICENSE.txt /licenses/licensing.txt

# Switch to manila user
USER manila
