
Dell EMC VNX Cinder driver containerization with OSP13
======================================================

Overview
--------
Dell EMC VNX Cinder driver integrates with the RedHat OpenStack Platform since OSP 6. In OSP13, all Cinder services are able to run within containers.
This instruction provides detailed steps on how to enable the containerization of VNX Cinder driver on top of the OSP Cinder images.


The VNX container image contains following RPM packages:

* EMC Naviseccli
* python-storops
* python-persist-queue
* python-cachez
* python-retryz


Deployment prerequisites
------------------------

* RedHat OpenStack Platform 13.
* VNX with Block version 5.32 or above.

Deployment steps
----------------

Prepare Dell EMC container
~~~~~~~~~~~~~~~~~~~~~~~~~~
The formal Dell EMC container image is published to `Red Hat Container Catalog <https://access.redhat.com/containers/>`_.

.. attention::

  in below examples, 192.168.139.1:8787 acts as a local registry.

Frist, pull the container image from Red Hat Container Catalog.

.. code-block:: bash

  $ docker pull registry.connect.redhat.com/dellemc/openstack-cinder-volume-dellemc

Then, tag and push it to the local docker registry if needed.

.. code-block:: bash

  $ docker tag registry.connect.redhat.com/dellemc/openstack-cinder-volume-dellemc 192.168.139.1:8787/dellemc/openstack-cinder-volume-dellemc
  $ docker push 192.168.139.1:8787/dellemc/openstack-cinder-volume-dellemc


Prepare custom environment yaml
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Define the customer docker registry.

*/home/stack/templates/custom-vnx-container.yaml*

.. code-block:: yaml

  parameter_defaults:

    DockerCinderVolumeImage: 192.168.139.1:8787/dellemc/openstack-cinder-volume-dellemc
    DockerInsecureRegistryAddress:
    - 192.168.139.1:8787

Above adds the director local registry IP `192.168.139.1:8787` to the `undercloud`.

- Define the VNX driver back end options.

The following sample environment file defines two VNX back ends, namely *vnx1* and *vnx2*:

*/home/stack/templates/custom-vnx-cinder.yaml*

.. code-block:: yaml

    parameter_defaults:
      CinderEnableIscsiBackend: false
      CinderEnableRbdBackend: false
      CinderEnableNfsBackend: false
      NovaEnableRbdBackend: false
      GlanceBackend: file
      ControllerExtraConfig:
        cinder::config::cinder_config:
            vnx1/volume_driver:
                value: cinder.volume.drivers.dell_emc.vnx.driver.VNXDriver
            vnx1/san_ip: #
                value: 192.168.1.50
            vnx1/san_login:
                value: admin
            vnx1/san_password:
                value: password
            vnx1/naviseccli_path:
                value: /opt/Navisphere/bin/naviseccli
            vnx1/initiator_auto_registration:
                value: True
            vnx1/storage_protocol:
                value: iscsi
            # second VNX backend
            vnx2/volume_driver:
                value: cinder.volume.drivers.dell_emc.vnx.driver.VNXDriver
            vnx2/san_ip:
                value: 192.168.1.50
            vnx2/san_login:
                value: admin
            vnx2/san_password:
                value: password
            vnx2/naviseccli_path:
                value: /opt/Navisphere/bin/naviseccli
            vnx2/initiator_auto_registration:
                value: True
            vnx2/storage_protocol:
                value: iscsi
        cinder_user_enabled_backends: ['vnx1','vnx2']

For a full detailed instruction of options, please refer to `VNX back end configuration <https://docs.openstack.org/cinder/pike/configuration/block-storage/drivers/emc-vnx-driver.html#back-end-configuration>`_

- Deploy the configured changes.

.. code-block:: bash

  (undercloud) $ openstack overcloud deploy --templates \
  -e /home/stack/templates/overcloud_images.yaml \
  -e /home/stack/templates/custom-vnx-container.yaml \
  -e /home/stack/templates/custom-vnx-cinder.yaml \
  -e <other templates>

The sequence of `-e` matters, Make sure the `/home/stack/templates/custom-vnx-container.yaml` appears after the `/home/stack/templates/overcloud_images.yaml`, so that
custom VNX container can be used instead of the default one.


- Verify the configured changes.

After the deployment finishes successfully, in the Cinder container, the `/etc/cinder/cinder.conf` should reflect the changes made above.

.. code-block:: ini

  ...
  enabled_backends=vnx1,vnx2
  ...
  [vnx1]
  initiator_auto_registration=True
  naviseccli_path=/opt/Navisphere/bin/naviseccli
  san_ip=192.168.1.50
  san_login=admin
  san_password=password
  storage_protocol=iscsi
  volume_driver=cinder.volume.drivers.dell_emc.vnx.driver.VNXDriver

  [vnx2]
  initiator_auto_registration=True
  naviseccli_path=/opt/Navisphere/bin/naviseccli
  san_ip=192.168.1.50
  san_login=admin
  san_password=password
  storage_protocol=iscsi
  volume_driver=cinder.volume.drivers.dell_emc.vnx.driver.VNXDriver

On the controller node, check the output of the Cinder container.

.. code-block:: bash

  $ tail -f /var/log/containers/cinder/cinder-volume.log
  2018-04-10 02:56:03.386 38 INFO storops.vnx.navi_command [req-ad774477-17d4-4579-8c89-bbcf5755af80 - - - - -] call command: /opt/Navisphere/bin/naviseccli -h 192.168.1.50 -user sysadmin -password *** -scope global -np connection -getport -all
