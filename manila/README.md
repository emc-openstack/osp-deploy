# Dell EMC Manila driver containerization with OSP16

## Overview

This instruction provides detailed steps on how to enable the containerization of VNX and Unity Manila driver on top of the OSP Manila images.

The Dell EMC Manila container image contains following packages:

- python-storops
- python-persist-queue
- python-cachez
- python-retryz

## VNX deployment

### Prerequisites

- Red Hat OpenStack Platform 16.
- VNX with File version 7.1 or above.

### Steps

Manila VNX driver is not depend on any third party libraries or tools, so there is no need to use Dell EMC customized docker image.

#### 1. Prepare custom environment yaml

Copy VNX configuration template `manila-vnx-config.yaml` from `tripleo-heat-templates`.

```bash
cp /usr/share/openstack-tripleo-heat-templates/environments/manila-vnx-config.yaml /home/stack/templates/
```

Modify backend configuration in `manila-vnx-config.yaml`.

```yaml
# This environment file enables Manila with the VNX backend.
resource_registry:
  OS::TripleO::Services::ManilaApi: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-api-container-puppet.yaml
  OS::TripleO::Services::ManilaScheduler: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-scheduler-container-puppet.yaml
  OS::TripleO::Services::ManilaShare: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-share-container-puppet.yaml
  OS::TripleO::Services::ManilaBackendUnity: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-backend-vnx.yaml

parameter_defaults:
  ManilaVNXBackendName: tripleo_manila_vnx
  ManilaVNXDriverHandlesShareServers: true
  ManilaVNXNasLogin: 'admin'
  ManilaVNXNasPassword: 'password'
  ManilaVNXNasServer: '192.168.1.49'
  ManilaVNXServerContainer: 'server_2'
  ManilaVNXShareDataPools: ''
  ManilaVNXEthernetPorts: 'cge-2-0'
```

IPv6 is supported by Manila VNX, parameter `ManilaIPv6` is used to enable IPv6:

```yaml
# This environment file enables Manila with the VNX backend.
resource_registry:
  OS::TripleO::Services::ManilaApi: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-api-container-puppet.yaml
  OS::TripleO::Services::ManilaScheduler: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-scheduler-container-puppet.yaml
  OS::TripleO::Services::ManilaShare: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-share-container-puppet.yaml
  OS::TripleO::Services::ManilaBackendUnity: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-backend-vnx.yaml

parameter_defaults:
  ManilaVNXBackendName: tripleo_manila_vnx
  ManilaVNXDriverHandlesShareServers: true
  ManilaVNXNasLogin: 'admin'
  ManilaVNXNasPassword: 'password'
  ManilaVNXNasServer: 'fd99:f17b:37d0::201'
  ManilaVNXServerContainer: 'server_2'
  ManilaVNXShareDataPools: ''
  ManilaVNXEthernetPorts: 'cge-2-0'
  ManilaIPv6: true
```

For a full detailed instruction of options, please refer to [VNX backend configuration](https://docs.openstack.org/manila/latest/configuration/shared-file-systems/drivers/dell-emc-vnx-driver.html)

#### 2. Deploy the configured changes

```bash
(undercloud) $ openstack overcloud deploy --templates \
  -e /home/stack/templates/containers-prepare-parameter.yaml \
  -e /home/stack/templates/manila-vnx-config.yaml \
  -e <other templates>
```

#### 3. Verify the configured changes

After the deployment finishes successfully, in the Manila container, the `/etc/manila/manila.conf` should reflect the changes made above.

```ini
[DEFAULT]
...
enabled_share_backends=tripleo_manila_vnx
...
[tripleo_manila_vnx]
share_driver=manila.share.drivers.emc.driver.EMCShareDriver
driver_handles_share_servers=True
emc_nas_login=admin
emc_nas_password=password
emc_nas_server=192.168.1.49
emc_share_backend=vnx
vnx_server_container=server_2
vnx_share_data_pools=
vnx_ethernet_ports=cge-2-0
network_plugin_ipv6_enabled=True
emc_ssl_cert_verify=False
```

## Unity deployment

### Prerequisites

- Red Hat OpenStack Platform 16.
- Unity with version 4.1 or above.

### Steps

#### 1. Prepare Dell EMC container

The formal Dell EMC container image is published to [Red Hat Container Catalog](https://access.redhat.com/containers/)

Red Hat OpenStack Platform supports remote registry and local registry for overcloud deployment. In this document, we only introduce local registry.

> in below examples, 192.168.139.1:8787 acts as a local registry.

**Notes:** We will not introduce how to setup local registry with docker-registry or docker-distribution. In RHOSP16,  undercloud also acts as a local registry, you could push Dell EMC container images to it, and pull these images when deploying overcloud, please refer to Red Hat document: [4.11. Undercloud container registry](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.0/html/director_installation_and_usage/installing-the-undercloud#undercloud-container-registry).

Frist, login registry.connect.redhat.com and pull the container image from Red Hat Container Catalog.

```bash
$ podman login -u username -p password registry.connect.redhat.com
$ podman pull registry.connect.redhat.com/dellemc/openstack-manila-share-dellemc-rhosp16
```

Then, tag and push it to the local registry.

```bash
$ podman tag registry.connect.redhat.com/dellemc/openstack-manila-share-dellemc-rhosp16 192.168.139.1:8787/dellemc/openstack-manila-share-dellemc-rhosp16
$ podman push 192.168.139.1:8787/dellemc/openstack-manila-share-dellemc-rhosp16
```

#### 2. Prepare custom environment yaml

##### 2.1 Define the custom docker registry

Create or edit `/home/stack/templates/custom-dellemc-container.yaml`.

```yaml
parameter_defaults:
  ContainerCinderVolumeImage: 192.168.139.1:8787/dellemc/openstack-manila-share-dellemc-rhosp16
  DockerInsecureRegistryAddress:
  - undercloud.ctlplane.localdomain:8787
  - 192.168.139.1:8787
```

**Notes:** Please add undercloud.ctlplane.localdomain:8787 as parameter of DockerInsecureRegistryAddress, otherwise overcloud will not able to pull images from undercloud.

##### 2.2 Prepare environment yaml for backend

Copy Unity configuration template `manila-unity-config.yaml` from `tripleo-heat-templates`.

```bash
cp /usr/share/openstack-tripleo-heat-templates/environments/manila-unity-config.yaml /home/stack/templates/
```

Modify backend configuration in `manila-unity-config.yaml`.

```yaml
# This environment file enables Manila with the Unity backend.
resource_registry:
  OS::TripleO::Services::ManilaApi: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-api-container-puppet.yaml
  OS::TripleO::Services::ManilaScheduler: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-scheduler-container-puppet.yaml
  OS::TripleO::Services::ManilaShare: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-share-container-puppet.yaml
  OS::TripleO::Services::ManilaBackendUnity: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-backend-unity.yaml

parameter_defaults:
  ManilaUnityBackendName: tripleo_manila_unity
  ManilaUnityDriverHandlesShareServers: true
  ManilaUnityNasLogin: 'admin'
  ManilaUnityNasPassword: 'password'
  ManilaUnityNasServer: '192.168.1.49'
  ManilaUnityServerMetaPool: 'pool_1'
  ManilaUnityShareDataPools: ''
  ManilaUnityEthernetPorts: ''
  ManilaUnityEmcSslCertVerify: false
  ManilaUnityEmcSslCertPath: ''
```

IPv6 is supported by Manila Unity, parameter `ManilaIPv6` is used to enable IPv6:

```yaml
# This environment file enables Manila with the Unity backend.
resource_registry:
  OS::TripleO::Services::ManilaApi: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-api-container-puppet.yaml
  OS::TripleO::Services::ManilaScheduler: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-scheduler-container-puppet.yaml
  OS::TripleO::Services::ManilaShare: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-share-container-puppet.yaml
  OS::TripleO::Services::ManilaBackendUnity: /usr/share/openstack-tripleo-heat-templates/deployment/manila/manila-backend-unity.yaml

parameter_defaults:
  ManilaUnityBackendName: tripleo_manila_unity
  ManilaUnityDriverHandlesShareServers: true
  ManilaUnityNasLogin: 'admin'
  ManilaUnityNasPassword: 'password'
  ManilaUnityNasServer: 'fd99:f17b:37d0::202'
  ManilaUnityServerMetaPool: 'pool_1'
  ManilaUnityShareDataPools: ''
  ManilaUnityEthernetPorts: ''
  ManilaUnityEmcSslCertVerify: false
  ManilaUnityEmcSslCertPath: ''
  ManilaIPv6: true
```

For a full detailed instruction of options, please refer to [Unity backend configuration](https://docs.openstack.org/manila/latest/configuration/shared-file-systems/drivers/dell-emc-unity-driver.html)

#### 3. Deploy the configured changes

```bash
(undercloud) $ openstack overcloud deploy --templates \
  -e /home/stack/templates/containers-prepare-parameter.yaml \
  -e /home/stack/templates/custom-dellemc-container.yaml \
  -e /home/stack/templates/manila-unity-config.yaml \
  -e <other templates>
```

The sequence of `-e` matters, Make sure the `/home/stack/templates/custom-vnx-container.yaml` appears after the `/home/stack/templates/containers-prepare-parameter.yaml`, so that custom VNX container can be used instead of the default one.

#### 4. Verify the configured changes

After the deployment finishes successfully, in the Manila container, the `/etc/manila/manila.conf` should reflect the changes made above.

```ini
[DEFAULT]
...
enabled_share_backends=tripleo_manila_unity
...
[tripleo_manila_unity]
share_driver=manila.share.drivers.emc.driver.EMCShareDriver
driver_handles_share_servers=True
emc_nas_login=admin
emc_nas_password=password
emc_nas_server=192.168.1.49
emc_share_backend=unity
unity_server_meta_pool=pool_1
unity_share_data_pools=
unity_ethernet_ports=
network_plugin_ipv6_enabled=True
emc_ssl_cert_verify=False
```
