..
     Licensed under the Apache License, Version 2.0 (the "License"); you may
     not use this file except in compliance with the License. You may obtain
     a copy of the License at
 
          http://www.apache.org/licenses/LICENSE-2.0
 
     Unless required by applicable law or agreed to in writing, software
     distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
     WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
     License for the specific language governing permissions and limitations
     under the License.

Cinder Volume Driver API / Developer Notes
==========================================

This document covers the Cinder volume driver API and issues that developers should be familiar with for driver development.

General Driver Issues
=====================

Driver Initialization
---------------------

.. code-block:: python

    __init__(self, execute=self.execute, *args, **kwargs)


The __init__ method is responsible for basic driver module initialization.

This method must succeed for the driver code to be loaded at all.  It should initialize all member variables that are used throughout the driver.


check_for_setup_error::
   check_for_setup_error(self)

This method is responsible for driver initialization that involves loading
configuration, talking to the backend storage array, etc.

If this method raises an exception, the driver will be left in an "uninitialized" state by the volume manager, which means that it will not be sent requests for volume operations.

This method typically checks things like whether the configured credentials can be used to log in the storage backend, and whether any external dependencies are present and working.

Commonly raised exceptions::
    VolumeBackendAPIException(data=message)


Volume Stats
------------

.. code-block:: python

    get_volume_stats(self, refresh=False)

The get_volume_stats method is used by the volume manager to collect information from the driver instance related to information about the driver, available and used space, and driver/backend capabilities.

It returns a dict with the following required fields:
  - volume_backend_name
    - This is an identifier for the backend taken from cinder.conf.  Useful when using multi-backend.
  - vendor_name
    - Vendor/author of the driver who serves as the contact for the driver's development and support.
  - driver_version
    - The driver version is logged at cinder-volume startup and is useful for tying volume service logs to a specific release of the code.  There are currently no rules for how or when this is updated, but it tends to follow typical major.minor.revision ideas.
  - storage_protocol
    - The protocol used to connect to the storage, this should be a short string such as: "iSCSI", "FC", "nfs", "ceph", etc.
  - total_capacity_gb
    - The total capacity in gigabytes (GiB) of the storage backend being used to store Cinder volumes.
  - free_capacity_gb
    - The free capacity in gigabytes (GiB).

And the following optional fields:
  - reserved_percentage  (integer)
    - Percentage of backend capacity which is not used by the scheduler.
  - location_info  (string)
    - Driver-specific information used by the driver and storage backend to correlate Cinder volumes and backend LUNs/files.
  - QoS_support  (Boolean)
    - Whether the backend supports quality of service.
  - provisioned_capacity_gb
    - The total provisioned capacity on the storage backend, in gigabytes (GiB), including space consumed by any user other than Cinder itself.
  - max_over_subscription_ratio
  - thin_provisioning_support  (Boolean)
    - Whether the backend is capable of allocating thinly provisioned volumes.
  - thick_provisioning_support  (Boolean)
    - Whether the backend is capable of allocating thick provisioned volumes.  (Typically True.)
  - total_volumes  (integer)
    - Total number of volumes on the storage backend.  This can be used in custom driver filter functions.
  - filter_function  (string)
    - A custom function used by the scheduler to determine whether a volume should be allocated to this backend or not.  Example::
      capabilities.total_volumes < 10
  - goodness_function  (string)
    - Similar to filter_function, but used to weigh multiple volume backends.  Example::
      capabilities.capacity_utilization < 0.6 ? 100 : 25
  - multiattach  (Boolean)
    - Whether the backend supports multiattach or not.  Defaults to False.
  - sparse_copy_volume  (Boolean)
    - Whether copies performed by the volume manager for operations such as migration should attempt to preserve sparseness.


The returned dict may also contain a list, "pools", which has a similar dict for each pool being used with the backend.


Driver Methods
==============

Create Volume
-------------
.. code-block:: python

    create_volume(self, volume)

Create a volume on the storage backend.

Returns:
  - dict of database updates for the new volume

This method is responsible only for storage allocation on the backend.

It should not export a LUN or actually make this storage available
for use, this is done in a later call.

Delete Volume
-------------
.. code-block:: python

    delete_volume(self, volume)

Delete a volume from the storage backend.

Prerequisites:
  - volume is not attached
  - volume has no snapshots

If the volume cannot be deleted from the backend, this call
typically fails with VolumeIsBusy or VolumeBackendAPIException.

If the driver can talk to the backend and detects that the volume
is no longer preset, this call should succeed and allow Cinder to
complete the process of deleting the volume.


Create Volume From Snapshot
---------------------------
.. code-block:: python

    create_volume_from_snapshot(self, volume, snapshot)

Create a new volume from a snapshot.

Prerequisites:
  - snapshot is not attached

Returns:
  - dict of database updates for the new volume

If the snapshot cannot be found, this call should succeed and allow
Cinder to complete the process of deleting the snapshot.


Clone Image
-----------
.. code-block:: python

    clone_image(self, volume, image_location, image_id,
                image_meta, image_service)

Create a volume efficiently from an image.

This method allows a driver to opt in to copying images in a more
efficient manner if they can be reached by the volume driver.

Returns:
  - a tuple of (update_dict, Boolean)
    where update_dict is a dictionary of database updates for the volume,
    and a Boolean indicating whether the clone occurred.

This method is optional.  If it is not implemented, the image is
copied onto the volume by attaching the image source and volume on
the volume service host and copying the data directly.  Drivers not
implementing this method should return::
  ({}, False)

The ability to return False indicating "did not succeed, fall back
to the standard copy method" means that drivers can implement this
in a best-effort fashion where it may only work if certain requirements
are met in the deployment.
