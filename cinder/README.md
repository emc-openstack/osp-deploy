# Dell EMC Cinder driver containerization with OSP16

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

#### Prepare Dell EMC container

The formal Dell EMC container image is published to [Red Hat Container Catalog](https://access.redhat.com/containers/)

Red Hat OpenStack Platform supports remote registry and local registry for overcloud deployment. In this document, we only introduce local registry.

> In below examples, 192.168.139.1:8787 acts as a local registry.

**Notes:** We will not introduce how to setup local registry with docker-registry or docker-distribution. In RHOSP16,  undercloud also acts as a local registry, you could push Dell EMC container images to it, and pull these images when deploying overcloud, please refer Red Hat documents.

Frist, login registry.connect.redhat.com and pull the container image from Red Hat Container Catalog.

```bash
$ docker login -u username -p password registry.connect.redhat.com
$ docker pull registry.connect.redhat.com/dellemc/openstack-cinder-volume-dellemc-rhosp16
```

Then, tag and push it to the local registry.

```bash
$ docker tag registry.connect.redhat.com/dellemc/openstack-cinder-volume-dellemc-rhosp16  192.168.139.1:8787/dellemc/openstack-cinder-volume-dellemc-rhosp16
$ docker push 192.168.139.1:8787/dellemc/openstack-cinder-volume-dellemc-rhosp16
```

#### Prepare custom environment yaml

##### Define the custom docker registry

Create or edit `/home/stack/templates/custom-dellemc-container.yaml`.

```yaml
parameter_defaults:
  ContainerCinderVolumeImage: 192.168.139.1:8787/dellemc/openstack-cinder-volume-dellemc-rhosp16
  DockerInsecureRegistryAddress:
  - undercloud.ctlplane.localdomain:8787
  - 192.168.139.1:8787
```

**Notes:** Please add undercloud.ctlplane.localdomain:8787 as parameter of DockerInsecureRegistryAddress, otherwise overcloud will not able to pull images from undercloud.

##### Prepare environment yaml for backend

Copy VNX configuration template `cinder-dellemc-vnx-config.yaml` from `tripleo-heat-templates`.

```bash
cp /usr/share/openstack-tripleo-heat-templates/environments/cinder-dellemc-vnx-config.yaml /home/stack/templates/
```

Modify backend configuration in `cinder-dellemc-vnx-config.yaml`.

```yaml
# A Heat environment file which can be used to enable a
# Cinder Dell EMC VNX backend, configured via puppet
resource_registry:
  OS::TripleO::Services::CinderBackendDellEMCVNX: /usr/share/openstack-tripleo-heat-templates/deployment/cinder/cinder-backend-dellemc-vnx-puppet.yaml

parameter_defaults:
  CinderEnableIscsiBackend: false
  CinderEnableRbdBackend: false
  CinderEnableNfsBackend: false
  NovaEnableRbdBackend: false
  GlanceBackend: file
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

#### Deploy the configured changes

```bash
(undercloud) $ openstack overcloud deploy --templates \
  -e /home/stack/templates/containers-prepare-parameter.yaml \
  -e /home/stack/templates/custom-dellemc-container.yaml \
  -e /home/stack/templates/cinder-backend-dellemc-vnx.yaml \
  -e <other templates>
```

The sequence of `-e` matters, Make sure the `/home/stack/templates/custom-vnx-container.yaml` appears after the `/home/stack/templates/containers-prepare-parameter.yaml`, so that custom VNX container can be used instead of the default one.

#### Verify the configured changes

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
volume_driver=cinder.volume.drivers.emc.vnx.driver.EMCVNXDriver
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

#### Prepare environment yaml for backend

Copy Unity configuration template `cinder-dellemc-unity-config.yaml` from `tripleo-heat-templates`.

```bash
cp /usr/share/openstack-tripleo-heat-templates/environments/cinder-dellemc-unity-config.yaml /home/stack/templates/
```

Modify backend configuration in `cinder-dellemc-unity-config.yaml`.

```yaml
# A Heat environment file which can be used to enable a
# Cinder Dell EMC Unity backend, configured via puppet
resource_registry:
  OS::TripleO::Services::CinderBackendDellEMCUnity: /usr/share/openstack-tripleo-heat-templates/deployment/cinder/cinder-backend-dellemc-unity-puppet.yaml

parameter_defaults:
  CinderEnableIscsiBackend: false
  CinderEnableRbdBackend: false
  CinderEnableNfsBackend: false
  NovaEnableRbdBackend: false
  GlanceBackend: file
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

#### Deploy the configured changes

```bash
(undercloud) $ openstack overcloud deploy --templates \
  -e /home/stack/templates/containers-prepare-parameter.yaml \
  -e /home/stack/templates/custom-dellemc-container.yaml \
  -e /home/stack/templates/cinder-backend-dellemc-unity.yaml \
  -e <other templates>
```

#### Verify the configured changes

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
