# Cinder driver container for Dell EMC VNX and Unity
# Using openstack-cinder-volume base image

FROM registry.redhat.io/rhosp15-rhel8/openstack-cinder-volume

ARG storops_version=1.2.2

MAINTAINER Dell EMC
LABEL description="Red Hat OpenStack Platform 15.0 cinder-volume Dell EMC VNX and Unity" \
      summary="Red Hat OpenStack Platform 15.0 cinder-volume Dell EMC VNX and Unity" \
      name="dellemc/openstack-cinder-volume-dellemc-rhosp15" \
      vendor="Dell EMC" \
      version="3.0.0" \
      storops=${storops_version} \
      naviseccli="7.33.9.1.55"

# Switch to root to install packages
USER root

# Download and install storops, naviseccli and dependencies
RUN curl -OL https://raw.githubusercontent.com/emc-openstack/storops/develop/utility/osp_deploy.sh
RUN sh osp_deploy.sh download ${storops_version}
RUN yum localinstall -y newpkgs/*.rpm
RUN rm -rf newpkgs
RUN rm -rf osp_deploy.sh

# Add license info
RUN curl -OL https://raw.githubusercontent.com/emc-openstack/osp-deploy/master/LICENSE.txt
RUN mkdir /licenses
RUN cp LICENSE.txt /licenses/licensing.txt

# Fix rpm verification issue
# Reference: https://bugzilla.redhat.com/show_bug.cgi?id=1560019
RUN chmod 0600 /etc/selinux/targeted/active/users_extra

# Switch to cinder user
USER cinder
