# EMC VNX FC driver

Copyright (c) 2012 - 2015 EMC Corporation, Inc.
All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License"); you ma
not use this file except in compliance with the License. You may obtain
a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
License for the specific language governing permissions and limitations
under the License.

## Overview

EMCCLIFCDriver (EMC VNX FC driver) is based on the FibreChannelDriver defined in Block Storage, with the ability to create/delete, attach/detach volumes, create/delete snapshots, etc. 

EMCCLIFCDriver performs the volume operations by executing Navisphere CLI. 

The Navisphere CLI (NaviSecCLI) is a Command Line Interface (CLI) used for management, diagnostics and reporting functions for VNX.

## Supported OpenStack Release

This driver supports Juno release.

## Requirements

* VNX OE for Block version 5.32 or higher.
* VNX Snapshot and Thin Provisioning license should be activated for VNX.
* Navisphere CLI v7.32 or higher is installed along with the driver.

## Supported Operations

The following operations are supported:

* Create, delete, attach and detach volumes
* Create and delete volume snapshots
* Create a volume from a snapshot
* Copy an image to a volume
* Clone a volume
* Extend a volume
* Migrate a volume
* Retype a volume
* Get volume statistics
* Create and delete consistency groups
* Create and delete consistency group snapshots

## Preparation

### Install Navisphere CLI 

Navisphere CLI needs to be installed on all Block Storage nodes within an OpenStack deployment.

* For Ubuntu x64, DEB is available in [EMC OpenStack Github](https://github.com/emc-openstack/naviseccli).
* For all other variants of Linux, Navisphere CLI is available at [Downloads for VNX2 Series](https://support.emc.com/downloads/36656_VNX2-Series) or [Downloads for VNX1 Series](https://support.emc.com/downloads/12781_VNX1-Series).
* After installation, set the security level of Navisphere CLI to low:

        /opt/Navisphere/bin/naviseccli security -certificate -setLevel low

### Install EMC VNX FC driver

EMC VNX FC driver provided in the installer package consists of 3 python files:

        emc_vnx_cli.py
        emc_cli_fc.py
        queueworker.py

Copy the 3 files above to the `cinder/volume/drivers/emc/` directory of the OpenStack node(s) where cinder-volume is running.

### FC zoning with VNX

FC zoning should be done between OpenStack node(s) and VNX by the storage administrator if FC SAN auto zoning is not enabled.

### Register with VNX

To access VNX storage, the compute nodes should be registered on VNX first if initiator auto registration is not enabled.

To perform "Copy Image to Volume" and "Copy Volume to Image" operations, the nodes running the cinder-volume service (Block Storage nodes) must be registered with the VNX as well.

The steps mentioned below are for the compute nodes. Please follow the same steps for the Block Storage nodes also (The steps can be skipped if initiator auto registration is enabled).

* Assume `20:00:00:24:FF:48:BA:C2:21:00:00:24:FF:48:BA:C2` is WWN of the FC initiator port name of the compute node. Register `20:00:00:24:FF:48:BA:C2:21:00:00:24:FF:48:BA:C2` in Unisphere
    * Login Unisphere, go to `FNM0000000000->Hosts->Initiators`,
    * Refresh and wait until initiator `20:00:00:24:FF:48:BA:C2:21:00:00:24:FF:48:BA:C2` with SP Port `A-1` appears.
    * Click the `Register` button, select `CLARiiON/VNX` and enter the host name (which is the output of the Linux command `hostname`) and IP address:
	    * Hostname : myhost1
	    * IP : 10.10.61.1
	    * Click Register. 
    * Now host 10.10.61.1 will appear under Hosts->Host List as well.
* Register the WWN with more ports if needed.

__Notes:__ When the driver notices that there is no existing storage group that has the host name as the storage group name, it will create the storage group and also add the compute node's or Block Storage nodes' registered initiators into the storage group.
If the driver notices that the storage group already exists, it will assume that the registered initiators have also been put into it and skip the operations above for better performance.
It is recommended that the storage administrator doesn't create the storage group manually and instead relies on the driver for the preparation. If the storage administrator do need to create the storage group manually for some special requirements, the correct registered initiators should be put into the storage group as well (otherwise the following volume attaching operations will fail).

## Backend configuration

Make the following changes in the configuration file `/etc/cinder/cinder.conf`:

The configuration options below are specific for EMC VNX FC driver

        storage_vnx_pool_name = Pool_01_SAS
        san_ip = 10.10.72.41
        san_secondary_ip = 10.10.72.42
        #VNX user name
        #san_login = username
        #VNX user password
        #san_password = password
        #VNX user type. Valid values are: global, local and ldap. global is the default value
        #storage_vnx_authentication_type = ldap
        #Directory path that contains the VNX security file. Make sure the security file is generated first
        #VNX credentials are not needed to configured with using security file
        storage_vnx_security_file_dir = /etc/secfile/array1
        naviseccli_path = /opt/Navisphere/bin/naviseccli
        # timeout in minutes
        default_timeout = 10
        volume_driver=cinder.volume.drivers.emc.emc_cli_fc.EMCCLIFCDriver
        destroy_empty_storage_group = False
        use_multi_iscsi_portals=False
        [database]
        max_pool_size=20
        max_overflow=30



* where `san_ip` is one of the SP IP address of the VNX array and `san_secondary_ip` is the other SP IP address of the VNX array. `san_ip` is a mandatory field which provides the primary connection to VNX array. And `san_secondary_ip` is an optional field which is for high availability (HA). In case that one SP is down, the other SP can be connected automatically. 
* where `Pool_01_SAS` is the pool from which the user wants to create volumes. The pools can be created using Unisphere for VNX. Refer to the "Multiple Pools Support" on how to manage multiple pools
* where `storage_vnx_security_file_dir` is the directory path of the VNX security file. Make sure the security file is generated following the steps in the authentication section
* Restart the cinder-volume service to make the configuration change take effect.

## Authentication

VNX credentials are necessary when the driver connects to the VNX system. Credentials in global, local and ldap scopes are supported. There are two approaches to provide the credentials:

The recommended one is using the Navisphere CLI security file to provide the credentials which can get rid of providing the plain text credentials in the configuration file. Following is the instruction on how to do this.

1. Find out the Linux user id of the `/usr/bin/cinder-volume` processes. Assuming the service /usr/bin/cinder-volume is running by the account `cinder`
2. Switch to root account

       $ sudo su
3. Change `cinder:x:113:120::/var/lib/cinder:/bin/false` to `cinder:x:113:120::/var/lib/cinder:/bin/bash` in `/etc/passwd` 
    (This temporary change is to make step 4 work.)
4. Save the credentials on behave of `cinder` user to a security file(assuming the array credentials are `admin/admin` in `global` scope). In the command below, the '-secfilepath' switch is used to specify the location to save the security file (assuming saving to directory /etc/secfile/array1).

        $ su -l cinder -c '/opt/Navisphere/bin/naviseccli -AddUserSecurity -user admin -password admin -scope 0 -secfilepath /etc/secfile/array1'
   Please save the security file to different locations for different arrays except where the same credentials are shared between all arrays managed by the host. Otherwise, the credentials in the security file will be overwritten. If `-secfilepath` is not specified in the command above, the security file will be saved to the default location which is the home directory of executor. 
5. Change `cinder:x:113:120::/var/lib/cinder:/bin/bash` back to `cinder:x:113:120::/var/lib/cinder:/bin/false` in /etc/passwd
6. Remove the credentials options `san_login`, `san_password` and `storage_vnx_authentication_type` from cinder.conf (normally it is `/etc/cinder/cinder.conf`). 
   Add option `storage_vnx_security_file_dir` and set its value to the directory path supplied with the '-secfilepath' switch in step 4. Omit this option if '-secfilepath' is not used in step 4.

        #Directory path that contains the VNX security file. Generate the security file first
        storage_vnx_security_file_dir = /etc/secfile/array1
7. Restart the cinder-volume service to make the change take effect

Alternatively, the credentials can be specified in /etc/cinder/cinder.conf by the three options below:

        #VNX user name
        san_login = username
        #VNX user password
        san_password = password
        #VNX user type. Valid values are: global, local and ldap. global is the default value
        storage_vnx_authentication_type = ldap

## Restrictions

* It does not suggest to deploy the driver on a compute node if "cinder upload-to-image --force True" is used against an in-use volume. Otherwise, "cinder upload-to-image --force True" will terminate the data access of the vm instance to the volume.
* VNX does not support to extend the thick volume which has a snapshot. If the user tries to extend a volume which has a snapshot, the status of the volume would change to "error_extending".
* VNX does not support to add compressed volumes into consistency groups.

## Provisioning type (thin, thick, deduplicated and compressed)

User can specify extra spec key `storagetype:provisioning` in volume type to set the provisioning type of a volume. The provisioning type can be thin, thick, deduplicated or compressed.

* **thin:**
  `thin` provisioning type means the volume is virtually provisioned

* **thick:**
  `thick` provisioning type means the volume is fully provisioned

* **deduplicated:**
  `deduplicated` provisioning type means the volume is virtually provisioned and the deduplication is enabled on it. The administrator shall go to VNX to configure the system level deduplication settings. To create a deduplicated volume, the VNX Deduplication license should be activated on VNX first, and use key deduplication_support=True to let Block Storage scheduler find a volume backend which manages a VNX with deduplication license activated

* **compressed:**
  `compressed` provisioning type means the volume is virtually provisioned and the compression is enabled on it. The administrator shall go to the VNX to configure the system level compression settings. To create a compressed volume, the VNX Compression license should be activated on VNX first, and use key compression_support=True to let Block Storage scheduler find a volume backend which manages a VNX with Compression license activated. VNX does not support to create a snapshot on a compressed volume. If the user tries to create a snapshot on a compressed volume, the operation would fail and OpenStack would show the new snapshot in error state

Here is an example about how to create a volume with provisioning type. First create a volume type and specify storage pool in the extra spec, then create a volume with this volume type.

        cinder type-create "ThickVolume"
        cinder type-create "ThinVolume"
        cinder type-create "DeduplicatedVolume"
        cinder type-create "CompressedVolume"
        cinder type-key "ThickVolume" set storagetype:provisioning=thick 
        cinder type-key "ThinVolume" set storagetype:provisioning=thin
        cinder type-key "DeduplicatedVolume" set storagetype:provisioning=deduplicated deduplication_support=True
        cinder type-key "CompressedVolume" set storagetype:provisioning=compressed compression_support=True

In the example above, four volume types are created: `ThickVolume`, `ThinVolume`, `DeduplicatedVolume` and `CompressedVolume`. For `ThickVolume`, `storagetype:provisioning` is set to `thick`. Similarly for the other volume types. If `storagetype:provisioning` is not specified or an invalid value, default value `thick` is adopted.

Volume type name, such as `ThickVolume`, is user-defined and can be any name. Extra spec key `storagetype:provisioning` shall be the exact name listed here. Extra spec value for `storagetype:provisioning` shall be `thick`, `thin`, `deduplicated` or `compressed`.
During volume creation, if the driver finds `storagetype:provisioning` in the extra spec of the volume type, it will create the volume of the provisioning type accordingly. Otherwise, the volume will be thick as default.

## Fully automated storage tiering support

VNX supports fully automated storage tiering which requires the FAST license activated on the VNX. The OpenStack administrator can use the extra spec key `storagetype:tiering` to set the tiering policy of a volume and use the key `fast_support=True` to let Block Storage scheduler find a volume backend which manages a VNX with FAST license activated. Here are the five supported values for the extra spec key `storagetype:tiering`:

        StartHighThenAuto (Default option)
        Auto
        HighestAvailable
        LowestAvailable
        NoMovement

Tiering policy can not be set for a deduplicated volume. The user can check storage pool properties on VNX to know the tiering policy of a deduplicated volume.

Here is an example about how to create a volume with tiering policy:

        cinder type-create "AutoTieringVolume"
        cinder type-key "AutoTieringVolume" set storagetype:tiering=Auto fast_support=True
        cinder type-create "ThinVolumeOnLowestAvaibleTier"
        cinder type-key "CompressedVolumeOnLowestAvaibleTier" set storagetype:provisioning=thin storagetype:tiering=Auto fast_support=True

## FAST Cache support

VNX has FAST Cache feature which requires the FAST Cache license activated on the VNX. The OpenStack administrator can use the extra spec key `fast_cache_enabled` to choose whether to create a volume in a storage pool with FAST Cache enabled.
The value of the extra spec key `fast_cache_enabled` is either `True` or `False`. When creating a volume, if the key `fast_cache_enabled=True` is set in the volume type, the volume will be created by a backend which manages at least a storage pool with FAST Cache enabled)

## Storage group automatic deletion

For volume attaching, the driver has a storage group on VNX for each compute node hosting the vm instances which are going to consume VNX Block Storage (using compute node's hostname as storage group's name). All the volumes attached to the vm instances in a computer node will be put into the storage group. If `destroy_empty_storage_group=True`, the driver will remove the empty storage group after its last volume is detached.
For data safety, it does not suggest to set `destroy_empty_storage_group=True` unless the VNX is exclusively managed by one Block Storage node because consistent `lock_path` is required for operation synchronization for this behavior.

## EMC storage-assisted volume migration

EMC VNX FC driver supports storage-assisted volume migration, when the user starts migrating with "cinder migrate --force-host-copy False <volume_id> <host>" or 
"cinder migrate <volume_id> <host>", cinder will try to leverage the VNX's native volume migration functionality.

In following scenarios, VNX storage-assisted volume migration will not be triggered:

1. Volume migration between backends with different storage protocol, ex, FC and iSCSI
2. Volume migration from pool-based backend to array-based backend
3. Volume is to be migrated across arrays

## Initiator auto registration

If `initiator_auto_registration=True`, the driver will automatically register FC initiators with all working FC target ports on the VNX array during volume attaching (The driver will skip those initiators that have already been registered)

If the user wants to register the initiators with some specific ports but not register with the other ports, this functionality should be disabled.

## Initiator auto deregistration

Enabling storage group automatic deletion is the precondition of this function. If `initiator_auto_deregistration=True` is set, the driver will deregister all the FC initiators of the host after its storage group is deleted.

## FC SAN auto zoning

EMC VNX FC driver supports FC SAN auto zoning when ZoneManager is configured. Set `zoning_mode` to `fabric` in `DEFAULT` section to enable this feature. For ZoneManager configuration, please refer to Block Storage official guide.

## Read-only volumes

OpenStack support read-only volumes. The following command can be used to set a volume as read-only.

        cinder --os-username admin --os-tenant-name admin readonly-mode-update <volume> True

After a volume is marked as read-only, the driver will forward the information when a hypervisor is attaching the volume and the hypervisor will have an implementation-specific way to make sure the volume is not written.

##  Volume number threshold

In VNX, there is limit on the maximum number of pool volumes that can be created in the system. When the limit is reached, no more pool volumes can be created even if there is remaining capacity in the storage pool. In other words, if the scheduler dispatches a volume creation request to a backend that has free capacity but reaches the limit, the backend will fail to create the corresponding volume.

The default value of `check_max_pool_luns_threshold` is `False`. When `check_max_pool_luns_threshold=True`, the pool-based backend will check the limit and will report 0 free capacity to the scheduler if the limit is reached. So the scheduler will be able to skip this kind of pool-based backend that runs out of pool volume number.

## Multiple pools support

When a storage pool is configured for a Block Storage backend (named as pool-based backend), only that storage pool will be used by that Block Storage backend.
If `storage_vnx_pool_name` is not configured, then all the pools on the VNX array will be used by that Block Storage backend and the scheduler will choose which pool to place the volume based on the capacities and capabilities of the pools.
This kind of Block Storage backend is named as array-based backend.

To adhere to the official pool-aware scheduler framework, the old extra spec key `storagetype:pool` is obsoleted and it will be ignored by the newer driver. `pool_name` is the replacement offered by pool-aware scheduler framework.

Here is an example about configuration of array-based backend.

        san_ip = 10.10.72.41
        #Directory path that contains the VNX security file. Make sure the security file is generated first
        storage_vnx_security_file_dir = /etc/secfile/array1
        storage_vnx_authentication_type = global
        naviseccli_path = /opt/Navisphere/bin/naviseccli
        default_timeout = 10
        volume_driver=cinder.volume.drivers.emc.emc_cli_iscsi.EMCCLIISCSIDriver
        destroy_empty_storage_group = False
        volume_backend_name = vnx_41

In this configuration, if the user wants to create a volume on a certain storage pool, a volume type with a extra spec specified storage pool should be created first, then the user can use this volume type to create the volume.

Here is an example about the volume type creation:

        cinder type-create "HighPerf"
        cinder type-key "HighPerf" set pool_name=Pool_02_SASFLASH volume_backend_name=vnx_41

## Multi-backend configuration

        [DEFAULT]

        enabled_backends=backendA, backendB

        [backendA]

        storage_vnx_pool_name = Pool_01_SAS
        san_ip = 10.10.72.41
        #Directory path that contains the VNX security file. Make sure the security file is generated first
        storage_vnx_security_file_dir = /etc/secfile/array1
        naviseccli_path = /opt/Navisphere/bin/naviseccli
        # Timeout in Minutes
        default_timeout = 10
        volume_driver=cinder.volume.drivers.emc.emc_cli_fc.EMCCLIFCDriver
        destroy_empty_storage_group = False
        initiator_auto_registration=True
        use_multi_iscsi_portals=False
        [backendB]
        storage_vnx_pool_name = Pool_02_SAS
        san_ip = 10.10.26.101
        san_login = username
        san_password = password
        naviseccli_path = /opt/Navisphere/bin/naviseccli
        # Timeout in Minutes
        default_timeout = 10
        volume_driver=cinder.volume.drivers.emc.emc_cli_fc.EMCCLIFCDriver
        destroy_empty_storage_group = False
        initiator_auto_registration=True
        use_multi_iscsi_portals=False
        [database]

        max_pool_size=20
        max_overflow=30

For more details on multi-backends, see [OpenStack Cloud Administration Guide](http://docs.openstack.org/admin-guide-cloud/content/multi_backend.html)

## Batch Processing for Volume Attaching/Detaching

Batch Processing is introduced to improve the performance of volume attaching/detaching. The driver accumulates the concurrent attaching/detaching requests and then serves the requests in batch later. Because some duplicated operations will be removed, the whole process will be more efficient. The minimum serving time of a request may increase since time is needed for requests to accumulate but the maximum serving time will be reduced as long as the time for accumulation is not excessively long. Batch processing is disabled by default.
Option `attach_detach_batch_interval` within the backend section is used to control this support.

* `attach_detach_batch_interval=-1`: Batch processing is disabled. This is the default value.

* `attach_detach_batch_interval=<Number of seconds>`: Batch processing is enabled and worker threads will sleep `<Number of seconds>` for the requests to accumulate before it serves them in batch.

## Force delete volumes in storage group

Some `available` volumes may remain in storage group on the VNX array due to some OpenStack timeout issue. But the VNX array do not allow the user to delete the volumes which are in storage group. Option `force_delete_lun_in_storagegroup` is introduced to allow the user to delete the `available` volumes in this tricky situation.

When `force_delete_lun_in_storagegroup=True` in the backend section, the driver will move the volumes out of storage groups and then delete them if the user tries to delete the volumes remain in storage group on the VNX array.

The default value of `force_delete_lun_in_storagegroup` is `False`.
