# Cinder driver container for Dell EMC VNX and Unity
# Using openstack-cinder-volume base image
# Version 1.0.0

FROM registry.access.redhat.com/rhosp12/openstack-cinder-volume:12.0-20180319.1

MAINTAINER Dell EMC
LABEL name="rhosp12/openstack-cinder-volume-dellemc"
LABEL vendor="Dell EMC"
LABEL version="1.0.0"
LABEL release="12.0-20180319.1"
LABEL storops="0.5.8"
LABEL naviseccli="7.33.9.1.55"

# Switch to root to install packages
USER root

# Download and install storops, naviseccli and dependencies
RUN curl -OL https://raw.githubusercontent.com/emc-openstack/storops/develop/utility/osp_deploy.sh
RUN sh osp_deploy.sh download 0.5.8
RUN yum localinstall -y newpkgs/*.rpm
RUN rm -rf newpkgs
RUN rm -rf osp_deploy.sh

# Switch to cinder user
USER cinder
