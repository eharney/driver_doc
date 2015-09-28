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
