Cinder Volume Driver API / Developer Notes
==========================================

This document covers the Cinder volume driver API and issues that developers should be familiar with for driver development.

General Driver Issues
=====================

Driver Initialization
---------------------

Init::
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
::
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
