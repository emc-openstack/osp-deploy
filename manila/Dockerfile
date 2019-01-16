# Manila driver container for Dell EMC Unity
# Using openstack-manila-share base image
# Version 1.1.2

FROM registry.access.redhat.com/rhosp13/openstack-manila-share

MAINTAINER Dell EMC
LABEL description="Red Hat OpenStack Platform 13.0 manila-share Dell EMC Unity" \
      summary="Red Hat OpenStack Platform 13.0 manila-share Dell EMC Unity" \
      name="rhosp13/openstack-manila-share-dellemc" \
      vendor="Dell EMC" \
      version="1.1.2" \
      storops="1.0.1"

# Switch to root to install packages
USER root

# Download and install storops and dependencies
RUN curl -OL https://raw.githubusercontent.com/emc-openstack/storops/develop/utility/osp_deploy.sh
RUN sh osp_deploy.sh download 1.0.1
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
