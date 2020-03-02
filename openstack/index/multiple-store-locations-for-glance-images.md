# Multiple store locations for glance images

In this post i will show you how to add multiple store locations for glance images. This will allow you to extend your glance capacity without affect your current stored images. The location can be any device or directory mounted at your glance host as a NFS, a physical hard disk, or an extended partition. Let's start:

First we need to create the directories where hard disks are going to be mounted

```text
sudo mkdir /var/lib/glance/lvm-images
sudo mkdir /var/lib/glance/extended-images
```

Next, we mount the devices at the directories created in the previous step

```text
sudo mount /dev/sdc1 /var/lib/glance/lvm-images/
sudo mount /dev/sdd1 /var/lib/glance/extended-images/
```

An important step is making the glance user the owner of that directories

```text
chown glance:glance /var/lib/glance/lvm-images/
chown glance:glance /var/lib/glance/extended-images/
```

Once the previous steps has been made, we need to configure the /etc/glance/glance-api.conf file.

In this file, we're going to configure glance to use multiple directories to store images. We search the section "Filesystem Store Options" and modify/create the following: 

We will leave the option "filesystem\_store\_datadir=" empty, if we comment this option, glance will use it as default store location and will show us an error during image creation. 

And we add the option "filesystem\_store\_datadirs", once for any directory we created in previous steps. We can use priorities on glance, priority 200 has precedence over priority 100, if we don't specify any priority, default will be 0

```text
# ============ Filesystem Store Options ========================
filesystem_store_datadir=
filesystem_store_datadirs=/var/lib/glance/images
filesystem_store_datadirs=/var/lib/glance/lvm-images:200
filesystem_store_datadirs=/var/lib/glance/extended-images:100
```

Once we have configured glance-api.conf, restart glance-api service

```text
$ glance image-create --name CirrosDatadir --file ~/Images/cirros-0.3.4-i386-disk.img --disk-format qcow2 --container-format bare --progress
[=============================>] 100%
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 79b4436412283bb63c2cba4ac796bcd9     |
| container_format | bare                                 |
| created_at       | 2015-08-13T11:34:00.000000           |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | 6ac8f5b9-5863-46ca-bb04-db352d35d829 |
| is_public        | False                                |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | CirrosDatadir                        |
| owner            | 738ec25d8b9c41f9b0cf84ce25730e92     |
| protected        | False                                |
| size             | 12506112                             |
| status           | active                               |
| updated_at       | 2015-08-13T11:34:09.000000           |
| virtual_size     | None                                 |
+------------------+--------------------------------------+
```

The image has been properly created at glance, we're going to check if the image has been properly created in the expected location. 

As we have configured a priority of 200 on this directory, the image must be here.

```text
$ ls -lsrt /var/lib/glance/extended-images/
total 12216 12216 -rw-r-----. 1 glance glance 12506112 ago 13 13:34 6ac8f5b9-5863-46ca-bb04-db352d35d829
```

We have to keep the store location that we have been using till now, the images remain available here.

```text
$ ls -lsrt /var/lib/glance/images/
total 12892 12892 -rw-r-----. 1 glance glance 13200896 ago 6 11:09 10a7a49f-2533-4513-881f-c4c6e419b778
```

Finally we check if the images are in active status

```text
glance image-list
+--------------------------------------+----------------+-------------+------------------+----------+--------+
| ID                                   | Name           | Disk Format | Container Format | Size     | Status |
+--------------------------------------+----------------+-------------+------------------+----------+--------+
| 10a7a49f-2533-4513-881f-c4c6e419b778 | cirros         | qcow2       | bare             | 13200896 | active |
| 6ac8f5b9-5863-46ca-bb04-db352d35d829 | CirrosDatadir  | qcow2       | bare             | 12506112 | active |
| 9e957bad-d0f8-4294-a438-77ad0d6af02b | CirrosDatadir2 | qcow2       | bare             | 12506112 | active |
+--------------------------------------+----------------+-------------+------------------+----------+--------+
```

Regards

