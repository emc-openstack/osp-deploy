# Dell EMC Unity and VNX Cinder driver containerization with OSP16

## Overview

This instruction provides detailed steps on how to enable the containerization of VNX and Unity Cinder driver on top of the OSP Cinder images.

The Dell EMC Cinder container image contains following packages:

- python-storops
- python-persist-queue
- python-cachez
- python-retryz
- Dell EMC Naviseccli

## VNX deployment

### Prerequisites

- Red Hat OpenStack Platform 16.
- VNX with Block version 5.32 or above.

### Steps

#### 1. Prepare Dell EMC container

The formal Dell EMC container image is published to [Red Hat Container Catalog](https://access.redhat.com/containers/)

Red Hat OpenStack Platform supports remote registry and local registry for overcloud deployment. In this document, we only introduce local registry.

> In below examples, 192.168.139.1:8787 acts as a local registry.

**Notes:** We will not introduce how to setup local registry with docker-registry or docker-distribution. In RHOSP16,  undercloud also acts as a local registry, you could push Dell EMC container images to it, and pull these images when deploying overcloud, please refer to Red Hat document: [4.11. Undercloud container registry](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.2/html/director_installation_and_usage/installing-the-undercloud#undercloud-container-registry).

Frist, login registry.connect.redhat.com and pull the container image from Red Hat Container Catalog.

```bash
$ podman login -u username -p password registry.connect.redhat.com
$ podman pull registry.connect.redhat.com/dellemc/openstack-cinder-volume-dellemc-rhosp16
```

Then, tag and push it to the local registry.

```bash
$ podman tag registry.connect.redhat.com/dellemc/openstack-cinder-volume-dellemc-rhosp16 192.168.139.1:8787/dellemc/openstack-cinder-volume-dellemc-rhosp16
$ podman push 192.168.139.1:8787/dellemc/openstack-cinder-volume-dellemc-rhosp16
```

#### 2. Prepare custom environment yaml

##### 2.1 Define the custom docker registry

Create or edit `/home/stack/templates/custom-dellemc-container.yaml`.

```yaml
parameter_defaults:
  ContainerCinderVolumeImage: 192.168.139.1:8787/dellemc/openstack-cinder-volume-dellemc-rhosp16
  DockerInsecureRegistryAddress:
  - undercloud.ctlplane.localdomain:8787
  - 192.168.139.1:8787
```

**Notes:** Please add undercloud.ctlplane.localdomain:8787 as parameter of DockerInsecureRegistryAddress, otherwise overcloud will not able to pull images from undercloud.

##### 2.2 Prepare environment yaml for backend

Copy VNX configuration template `cinder-dellemc-vnx-config.yaml` from `tripleo-heat-templates`.

```bash
cp /usr/share/openstack-tripleo-heat-templates/environments/cinder-dellemc-vnx-config.yaml /home/stack/templates/
```

Modify backend configuration in `cinder-dellemc-vnx-config.yaml`.

Note: **LVM driver** is enabled by default in TripleO, you want to set the ```CinderEnableIscsiBackend``` to false in one of your environment file to turn it off.

```yaml
parameter_defaults:
  CinderEnableIscsiBackend: false
```

```yaml
# A Heat environment file which can be used to enable a
# Cinder Dell EMC VNX backend, configured via puppet
resource_registry:
  OS::TripleO::Services::CinderBackendDellEMCVNX: /usr/share/openstack-tripleo-heat-templates/deployment/cinder/cinder-backend-dellemc-vnx-puppet.yaml

parameter_defaults:
  CinderEnableDellEMCVNXBackend: true
  CinderDellEMCVNXBackendName: 'tripleo_dellemc_vnx'
  CinderDellEMCVNXSanIp: '192.168.1.50'
  CinderDellEMCVNXSanLogin: 'admin'
  CinderDellEMCVNXSanPassword: 'password'
  CinderDellEMCVNXStorageProtocol: 'iscsi'
  CinderDellEMCVNXStoragePoolName: ''
  CinderDellEMCVNXDefaultTimeout: 3600
  CinderDellEMCVNXMaxLunsPerStorageGroup: 255
  CinderDellEMCVNXInitiatorAutoRegistration: 'true'
  CinderDellEMCVNXAuthType: 'global'
  CinderDellEMCVNXStorageSecurityFileDir: ''
  CinderDellEMCVNXNaviSecCliPath: '/opt/Navisphere/bin/naviseccli'
```

For a full detailed instruction of options, please refer to [VNX backend configuration](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/dell-emc-vnx-driver.html)

#### 3. Deploy the configured changes

```bash
(undercloud) $ openstack overcloud deploy --templates \
  -e /home/stack/templates/containers-prepare-parameter.yaml \
  -e /home/stack/templates/custom-dellemc-container.yaml \
  -e <other templates> \
  -e /home/stack/templates/cinder-backend-dellemc-vnx.yaml
```

The sequence of `-e` matters, Make sure the `/home/stack/templates/custom-vnx-container.yaml` appears after the `/home/stack/templates/containers-prepare-parameter.yaml`, so that custom VNX container can be used instead of the default one.

#### 4. Verify the configured changes

After the deployment finishes successfully, in the Cinder container, the `/etc/cinder/cinder.conf` should reflect the changes made above.

```ini
[DEFAULT]
...
enabled_backends=tripleo_dellemc_vnx
...
[tripleo_dellemc_vnx]
default_timeout=3600
max_luns_per_storage_group=255
naviseccli_path=/opt/Navisphere/bin/naviseccli
san_ip=192.168.1.50
san_login=admin
san_password=password
storage_vnx_pool_names=
volume_backend_name=tripleo_dellemc_vnx
volume_driver=cinder.volume.drivers.dell_emc.vnx.driver.VNXDriver
storage_protocol=iscsi
initiator_auto_registration=True
storage_vnx_authentication_type=global
storage_vnx_security_file_dir=
```

## Unity deployment

### Prerequisites

- Red Hat OpenStack Platform 16.
- Unity with version 4.1 or above.

### Steps

Unity is almost same as VNX, the only difference is you need to prepare Unity specific yaml.

#### 1. Prepare environment yaml for backend

Copy Unity configuration template `cinder-dellemc-unity-config.yaml` from `tripleo-heat-templates`.

```bash
cp /usr/share/openstack-tripleo-heat-templates/environments/cinder-dellemc-unity-config.yaml /home/stack/templates/
```

Modify backend configuration in `cinder-dellemc-unity-config.yaml`.

Note: **LVM driver** is enabled by default in TripleO, you want to set the ```CinderEnableIscsiBackend``` to false in one of your environment file to turn it off.

```yaml
parameter_defaults:
  CinderEnableIscsiBackend: false
```

```yaml
# A Heat environment file which can be used to enable a
# Cinder Dell EMC Unity backend, configured via puppet
resource_registry:
  OS::TripleO::Services::CinderBackendDellEMCUnity: /usr/share/openstack-tripleo-heat-templates/deployment/cinder/cinder-backend-dellemc-unity-puppet.yaml

parameter_defaults:
  CinderEnableDellEMCUnityBackend: true
  CinderDellEMCUnityBackendName: 'tripleo_dellemc_unity'
  CinderDellEMCUnitySanIp: '192.168.1.50'
  CinderDellEMCUnitySanLogin: 'admin'
  CinderDellEMCUnitySanPassword: 'password'
  CinderDellEMCUnityStorageProtocol: 'iSCSI'
  CinderDellEMCUnityIoPorts: ''
  CinderDellEMCUnityStoragePoolNames: ''
```

For a full detailed instruction of options, please refer to [Unity backend configuration](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/dell-emc-unity-driver.html)

#### 2. Deploy the configured changes

```bash
(undercloud) $ openstack overcloud deploy --templates \
  -e /home/stack/templates/containers-prepare-parameter.yaml \
  -e /home/stack/templates/custom-dellemc-container.yaml \
  -e <other templates> \
  -e /home/stack/templates/cinder-backend-dellemc-unity.yaml
```

#### 3. Verify the configured changes

After the deployment finishes successfully, in the Cinder container, the `/etc/cinder/cinder.conf` should reflect the changes made above.

```ini
[DEFAULT]
...
enabled_backends=tripleo_dellemc_unity
...
[tripleo_dellemc_unity]
volume_backend_name=tripleo_dellemc_unity
volume_driver=cinder.volume.drivers.dell_emc.unity.Driver
san_ip=192.168.1.50
san_login=admin
san_password=password
storage_protocol=iSCSI
unity_io_ports=
unity_storage_pool_names=
```

## Multiple Back-ends

To configure multiple backends of two different storage types you can combine pass both environment files on the same deploy command as below.

`cinder-dellemc-unity-config.yaml` and `cinder-dellemc-vnx-config.yaml` can be used to configure both unity and vnx as multi-backends.

```bash
(undercloud) $ openstack overcloud deploy --templates \
  -e /home/stack/templates/containers-prepare-parameter.yaml \
  -e /home/stack/templates/custom-dellemc-container.yaml \
  -e <other templates> \
  -e /home/stack/templates/cinder-backend-dellemc-vnx.yaml \
  -e /home/stack/templates/cinder-backend-dellemc-unity.yaml
```


## Multiple Back-ends of the same storage backend

`cinder-dellemc-unity-config.yaml` and `cinder-dellemc-vnx-config.yaml` cannot be used to configure multiple instances of the
same storage backend.

In order to configure multiple back-ends of same storage, we could use `ControllerExtraConfig` to set the configurations for all backends directly.

This is an example to enable 4 backends.

### Steps

#### 1. Prepare environment yaml for custom docker registry and multiple backends

Create or edit `/home/stack/templates/custom-dellemc-container.yaml`.

Note: **LVM driver** is enabled by default in TripleO, you want to set the ```CinderEnableIscsiBackend``` to false in one of your environment file to turn it off.
```yaml
parameter_defaults:
  CinderEnableIscsiBackend: false
```

```yaml
parameter_defaults:
  ContainerCinderVolumeImage: 192.168.139.1:8787/dellemc/openstack-cinder-volume-dellemc-rhosp16
  DockerInsecureRegistryAddress:
  - undercloud.ctlplane.localdomain:8787
  - 192.168.139.1:8787

  ControllerExtraConfig:
    # in this exapmle, we enable 4 backends
    cinder_user_enabled_backends: ['unity_fc', 'unity_iscsi', 'vnx_fc', 'vnx_iscsi']
    cinder::config::cinder_config:
      unity_fc/volume_driver:
        value: cinder.volume.drivers.dell_emc.unity.driver.UnityDriver
      unity_fc/volume_backend_name:
        value: unity_fc
      unity_fc/san_ip:
        value: <unity array sp ip>
      unity_fc/san_login:
        value: admin
      unity_fc/san_password:
        value: xxxxxx
      unity_fc/storage_protocol:
        value: FC
      unity_fc/unity_storage_pool_names:
        value: pool1
      unity_fc/default_timeout:
        value: 120
      unity_fc/max_over_subscription_ratio:
        value: 20
      unity_fc/use_multipath_for_image_xfer:
        value: True
      unity_fc/enforce_multipath_for_image_xfer:
        value: True
      unity_fc/image_volume_cache_enabled:
        value: True
      unity_fc/suppress_requests_ssl_warnings:
        value: True
      unity_iscsi/volume_driver:
        value: cinder.volume.drivers.dell_emc.unity.driver.UnityDriver
      unity_iscsi/volume_backend_name:
        value: unity_iscsi
      unity_iscsi/san_ip:
        value: <unity array sp ip>
      unity_iscsi/san_login:
        value: admin
      unity_iscsi/san_password:
        value: xxxxxx
      unity_iscsi/storage_protocol:
        value: iSCSI
      unity_iscsi/unity_storage_pool_names:
        value: pool1
      unity_iscsi/default_timeout:
        value: 120
      unity_iscsi/max_over_subscription_ratio:
        value: 20
      unity_iscsi/use_multipath_for_image_xfer:
        value: True
      unity_iscsi/enforce_multipath_for_image_xfer:
        value: True
      unity_iscsi/unity_io_ports:
        value: ['spa_iom_0_eth0','spb_iom_0_eth0']
      unity_iscsi/image_volume_cache_enabled:
        value: True
      unity_iscsi/suppress_requests_ssl_warnings:
        value: True
      vnx_fc/volume_driver:
        value: cinder.volume.drivers.dell_emc.vnx.driver.VNXDriver
      vnx_fc/volume_backend_name:
        value: vnx_fc
      vnx_fc/san_ip:
        value: <vnx array sp ip>
      vnx_fc/san_login:
        value: xxxxx
      vnx_fc/san_password:
        value: xxxxx
      vnx_fc/storage_protocol:
        value: FC
      vnx_fc/unity_storage_pool_names:
        value: pool1
      vnx_fc/default_timeout:
        value: 20
      vnx_fc/use_multipath_for_image_xfer:
        value: True
      vnx_fc/enforce_multipath_for_image_xfer:
        value: True
      vnx_fc/force_delete_lun_in_storagegroup:
        value: True
      vnx_fc/ignore_pool_full_threshold:
        value: True
      vnx_fc/image_volume_cache_enabled:
        value: True
      vnx_fc/initiator_auto_registration:
        value: True
      vnx_fc/force_delete_lun_in_storagegroup:
        value: True
      vnx_fc/vnx_async_migrate:
        value: False
      vnx_fc/destroy_empty_storage_group:
        value: False
      vnx_fc/num_volume_device_scan_tries:
        value: 10
      vnx_fc/naviseccli_path:
        value: /opt/Navisphere/bin/naviseccli
      vnx_fc/driver_use_ssl:
        value: True
      vnx_fc/driver_ssl_cert_verify:
        value: True
      vnx_fc/driver_ssl_cert_path:
        value: <path to cert>
      vnx_fc/suppress_requests_ssl_warnings:
        value: True
      vnx_iscsi/volume_driver:
        value: cinder.volume.drivers.dell_emc.vnx.driver.VNXDriver
      vnx_iscsi/volume_backend_name:
        value: vnx_iscsi
      vnx_iscsi/san_ip:
        value: <vnx array sp ip>
      vnx_iscsi/san_login:
        value: xxxxx
      vnx_iscsi/san_password:
        value: xxxx
      vnx_iscsi/storage_protocol:
        value: iSCSI
      vnx_iscsi/unity_storage_pool_names:
        value: pool1
      vnx_iscsi/default_timeout:
        value: 20
      vnx_iscsi/use_multipath_for_image_xfer:
        value: True
      vnx_iscsi/enforce_multipath_for_image_xfer:
        value: True
      vnx_iscsi/force_delete_lun_in_storagegroup:
        value: True
      vnx_iscsi/ignore_pool_full_threshold:
        value: True
      vnx_iscsi/image_volume_cache_enabled:
        value: True
      vnx_iscsi/initiator_auto_registration:
        value: True
      vnx_iscsi/force_delete_lun_in_storagegroup:
        value: True
      vnx_iscsi/vnx_async_migrate:
        value: False
      vnx_iscsi/destroy_empty_storage_group:
        value: False
      vnx_iscsi/suppress_requests_ssl_warnings:
        value: True
      vnx_iscsi/num_volume_device_scan_tries:
        value: 10
      vnx_iscsi/naviseccli_path:
        value: /opt/Navisphere/bin/naviseccli
      vnx_iscsi/driver_use_ssl:
        value: True
      vnx_iscsi/driver_ssl_cert_verify:
        value: True
      vnx_iscsi/driver_ssl_cert_path:
        value: <path to cert file>
      vnx_iscsi/io_port_list:
        value: ['a-5-0','b-5-0']
```

#### 2. Deploy the configured changes

```bash
(undercloud) $ openstack overcloud deploy --templates \
  -e /home/stack/templates/containers-prepare-parameter.yaml \
  -e <other templates> \
  -e /home/stack/templates/custom-dellemc-container.yaml
```
