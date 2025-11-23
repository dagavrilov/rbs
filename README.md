# Fault-tolerant Cinder volume driver
OpenStack Cinder Fault-Tolerant Driver based on LVM

# Features
- Create a fault-tolerant OpenStack Cinder storage resource/service with data replication on backup hosts.
- Disconnect block device volumes to one of the backup storage nodes in the event of a primary storage node failure.
- Freeze - disables block device lifecycle management operations while maintaining data access during a disaster.
- Restore to the original state, ensuring operations on the backup device as soon as it becomes available/functional.
- Thaw - completely unlocks block device lifecycle management operations.
- Thin and thick volumes
- Create, delete, mount, and dismount volumes.
- Create, view, and delete volume frames.
- Create a volume from a volume snapshot.
- Copy a machine behavior image to a volume.
- Copy a volume in a machine behavior image.
- Volume cloning.
- Migrating a volume between storage hosts
- Changing the block device type (reprint)
- Resizing a volume
- Compatibility with Python 3

# Driver installation
```
sudo apt install lvm2 targetcli-fb python3-rtslib-fb drbd-utils -y

git clone https://github.com/dagavrilov/ebs.git ebs 
sudo cp ebs/etc/cinder/rootwrap.d/ebs.filters /etc/cinder/rootwrap.d/
sudo cp -r ebs/cinder/volume/drivers/ovt /usr/lib/python3/dist-packages/cinder/volume/drivers/
```

# Openstack Cinder block device type with replication support creation
```
openstack volume type create ebs --property volume_backend_name='ebs' --property replication_enabled='<is> True'
```

# Example of Cinder volume configuration
```
[EBS]
target_helper=lioadm
target_protocol=iscsi 
target_ip_address=10.0.251.21
target_secondary_ip_addresses=10.0.252.21

volume_backend_name=ebs
volume_driver = cinder.volume.drivers.ovt.ebs.EBSVolumeDriver
volume_group=volumes

# storage backend id
backend_id=hci-0001@EBS
# storage API address
backend_ip=10.0.10.21
# storage API port
backend_port=7000

# available modes
# async : write completion is determined when data
# semi-sync: write completion is determined when data is written to the local disk and the local TOP transmission buffer
# full-sync: write completion is determined when data is written to both the local disk and the remote disk (default mode)
replication_mode = full-sync

#replication_resync_rate = 100
#replication_starting_port = 7001
replication_device = backend_id:hci-0002@EBS,ip:10.0.10.22,port:7000,volume_group:volumes
```
# Usage
Failover policy creation. The backup host (in the example, hci-0002@EBS) will be shut down and marked as failed-over, while the volumes on them will remain accessible:
```
openstack volume service list --long
+------------------+--------------+------+---------+-------+----------------------------+---------------------------------------------------------------+
| Binary           | Host         | Zone | Status  | State | Updated At                 | Disabled Reason                                               |
+------------------+--------------+------+---------+-------+----------------------------+---------------------------------------------------------------+
| cinder-scheduler | hci-0001     | AZ01 | enabled | up    | 2025-11-18T12:41:31.000000 | None                                                          |
| cinder-scheduler | hci-0002     | AZ01 | enabled | up    | 2025-11-18T12:41:33.000000 | None                                                          |
| cinder-scheduler | hci-0003     | AZ01 | enabled | up    | 2025-11-18T12:41:31.000000 | None                                                          |
| cinder-volume    | hci-0001@EBS | AZ01 | enabled | up    | 2025-11-18T12:41:27.000000 | None                                                          |
| cinder-volume    | hci-0002@EBS | AZ01 | enabled | up    | 2025-11-18T12:41:25.000000 | None                                                          |
+------------------+--------------+------+---------+-------+----------------------------+---------------------------------------------------------------+

cinder failover-host hci-0002@EBS --backend_id hci-0001@EBS

openstack volume service list --long
+------------------+--------------+------+----------+-------+----------------------------+---------------------------------------------------------------+
| Binary           | Host         | Zone | Status   | State | Updated At                 | Disabled Reason                                               |
+------------------+--------------+------+----------+-------+----------------------------+---------------------------------------------------------------+
| cinder-scheduler | hci-0001     | AZ01 | enabled  | up    | 2025-11-18T12:42:51.000000 | None                                                          |
| cinder-scheduler | hci-0002     | AZ01 | enabled  | up    | 2025-11-18T12:42:53.000000 | None                                                          |
| cinder-scheduler | hci-0003     | AZ01 | enabled  | up    | 2025-11-18T12:42:31.000000 | None                                                          |
| cinder-volume    | hci-0001@EBS | AZ01 | enabled  | up    | 2025-11-18T12:42:28.000000 | None                                                          |
| cinder-volume    | hci-0002@EBS | AZ01 | disabled | up    | 2025-11-18T12:42:24.000000 | failed-over                                                   |
+------------------+--------------+------+----------+-------+----------------------------+---------------------------------------------------------------+
```
