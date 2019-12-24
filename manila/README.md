# Dell EMC Manila driver containerization with OSP15

## Overview

This instruction provides detailed steps on how to enable the containerization of VNX and Unity Manila driver on top of the OSP Manila images.

The Dell EMC Manila container image contains following RPM packages:

- python-storops
- python-persist-queue
- python-cachez
- python-retryz

## VNX deployment

### Prerequisites

- Red Hat OpenStack Platform 15.
- VNX with File version 7.1 or above.

### Steps

Manila VNX driver is not depend on any third party libraries or tools, so there is no need to use Dell EMC customized docker image.

#### Prepare custom environment yaml

Copy VNX configuration template `manila-vnx-config.yaml` from `tripleo-heat-templates`.

```bash
cp /usr/share/openstack-tripleo-heat-templates/environments/manila-vnx-config.yaml /home/stack/templates/
```

Modify backend configuration in `manila-vnx-config.yaml`.

```yaml
# This environment file enables Manila with the VNX backend.
resource_registry:
  OS::TripleO::Services::ManilaBackendVNX: /usr/share/openstack-tripleo-heat-templates/puppet/services/manila-backend-vnx.yaml

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
  OS::TripleO::Services::ManilaBackendVNX: /usr/share/openstack-tripleo-heat-templates/puppet/services/manila-backend-vnx.yaml

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

#### Deploy the configured changes

```bash
(undercloud) $ openstack overcloud deploy --templates \
  -e /home/stack/templates/overcloud_images.yaml \
  -e /home/stack/templates/manila-vnx-config.yaml \
  -e <other templates>
```

#### Verify the configured changes

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

- Red Hat OpenStack Platform 15.
- Unity with version 4.1 or above.

### Steps

#### Prepare Dell EMC container

The formal Dell EMC container image is published to [Red Hat Container Catalog](https://access.redhat.com/containers/)

Red Hat OpenStack Platform supports remote registry and local registry for overcloud deployment. In this document, we only introduce local registry.

> in below examples, 192.168.139.1:8787 acts as a local registry.

Frist, login registry.connect.redhat.com and pull the container image from Red Hat Container Catalog.

```bash
$ docker login -u username -p password registry.connect.redhat.com
$ docker pull registry.connect.redhat.com/dellemc/openstack-manila-share-dellemc
```

Then, tag and push it to the local registry.

```bash
$ docker tag registry.connect.redhat.com/dellemc/openstack-manila-share-dellemc 192.168.139.1:8787/dellemc/openstack-manila-share-dellemc
$ docker push 192.168.139.1:8787/dellemc/openstack-manila-share-dellemc
```

#### Prepare custom environment yaml

##### Define the custom docker registry

Create or edit `/home/stack/templates/custom-dellemc-container.yaml`.

```yaml
parameter_defaults:
  DockerManilaShareImage: 192.168.139.1:8787/dellemc/openstack-manila-share-dellemc
  DockerInsecureRegistryAddress:
  - 192.168.139.1:8787
```

Above adds the director local registry IP `192.168.139.1:8787` to the `undercloud`.

##### Prepare environment yaml for backend

Copy Unity configuration template `manila-unity-config.yaml` from `tripleo-heat-templates`.

```bash
cp /usr/share/openstack-tripleo-heat-templates/environments/manila-unity-config.yaml /home/stack/templates/
```

Modify backend configuration in `manila-unity-config.yaml`.

```yaml
# This environment file enables Manila with the Unity backend.
resource_registry:
  OS::TripleO::Services::ManilaBackendUnity: /usr/share/openstack-tripleo-heat-templates/puppet/services/manila-backend-unity.yaml

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
  OS::TripleO::Services::ManilaBackendUnity: /usr/share/openstack-tripleo-heat-templates/puppet/services/manila-backend-unity.yaml

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

#### Deploy the configured changes

```bash
(undercloud) $ openstack overcloud deploy --templates \
  -e /home/stack/templates/overcloud_images.yaml \
  -e /home/stack/templates/custom-dellemc-container.yaml \
  -e /home/stack/templates/manila-unity-config.yaml \
  -e <other templates>
```

The sequence of `-e` matters, Make sure the `/home/stack/templates/custom-vnx-container.yaml` appears after the `/home/stack/templates/overcloud_images.yaml`, so that custom VNX container can be used instead of the default one.

#### Verify the configured changes

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
