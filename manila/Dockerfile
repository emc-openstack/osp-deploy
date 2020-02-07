# Manila driver container for Dell EMC Unity
# Using openstack-manila-share base image

FROM registry.access.redhat.com/rhosp14/openstack-manila-share

ARG storops_version=1.2.3

MAINTAINER Dell EMC
LABEL description="Red Hat OpenStack Platform 14.0 manila-share Dell EMC Unity" \
      summary="Red Hat OpenStack Platform 14.0 manila-share Dell EMC Unity" \
      name="dellemc/openstack-manila-share-dellemc-rhosp14" \
      vendor="Dell EMC" \
      version="2.0.1" \
      storops=${storops_version}

# Switch to root to install packages
USER root

# Download and install storops and dependencies
RUN curl -OL https://raw.githubusercontent.com/emc-openstack/storops/master/utility/osp_deploy.sh
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

# Switch to manila user
USER manila
