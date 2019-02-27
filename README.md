# Overview

Document Referenced in this project

Docker Container
> https://github.com/gluster/gluster-containers

Admin Guide
> https://gluster.readthedocs.io/en/latest/Administrator%20Guide/overview/

Official Volume Setup
> https://gluster.readthedocs.io/en/latest/Administrator%20Guide/Setting%20Up%20Volumes/

# Quick Started

Please execute the following commands one by one in CLI:

```
docker-compose up -d glusterfs1 glusterfs2 glusters3
echo "Sleep 10 seconds to wait the glusterfs up"
sleep 10
docker-compose exec glusterfs1 gluster peer probe glusterfs2
docker-compose exec glusterfs1 gluster volume create test-volume replica 2 transport tcp glusterfs1:/data/glusterfs/test glusterfs2:/data/glusterfs/test
docker-compose exec glusterfs1 gluster volume start test-volume
docker-compose exec glusterfs1 setfacl -m u:1000:rwx /data/glusterfs/test
docker-compose exec glusterfs2 setfacl -m u:1000:rwx /data/glusterfs/test
docker-compose exec glusterfs1 tail -f /var/log/glusterfs/bricks/data-glusterfs-test.log
```

In the next section, we will explain the command in the script.

# Explain

## Official Command Document Reference

> https://gluster.readthedocs.io/en/latest/CLI-Reference/cli-main/

### Peer Commands

The peer commands are used to manage the Trusted Server Pool (TSP).

Command|Syntax|Description
---|---|---
peer probe |peer probe server |Add server to the TSP
peer detach |peer detach server |Remove server from the TSP
peer status |peer status |Display the status of all nodes in the TSP
pool list |pool list |List all nodes in the TSP


### Volume Commands

The volume commands are used to setup and manage Gluster volumes.

Command |Syntax|Description
---|---|---
volume create |volume create volname [options] bricks |Create a volume called volname using the specified bricks with the configuration specified by options
volume start |volume start volname [force] |Start volume volname
volume stop |volume stop volname |Stop volume volname
volume info |volume info [volname] |Display volume info for volname if provided, else for all volumes on the TSP
volume status |volumes status[volname] |Display volume status for volname if provided, else for all volumes on the TSP
volume list |volume list |List all volumes in the TSP
volume set |volume set volname option value |Set option to value for volname
volume get |volume get volname <option,all> |Display the value of option (if specified)for volname , or all options otherwise
volume add-brick |volume add-brick brick-1 ... brick-n |Expand volname to include the bricks brick-1 to brick-n
volume remove-brick |volume remove-brick brick-1 ... brick-n <start,stop,status,commit,force> |Shrink volname by removing the bricks brick-1 to brick-n . start will trigger a rebalance to migrate data from the removed bricks. stop will stop an ongoing remove-brick operation. force will remove the bricks immediately and any data on them will no longer be accessible from Gluster clients.
volume replace-brick |volume replace-brick volname old-brick new-brick |Replace old-brick of volname with new-brick
volume delete |volume delete volname |Delete volname

## Usage Sample

```
$> gluster pool list
UUID                                    Hostname        State
3c510d0e-bc1d-4bf6-b41b-5bd4632b0d8a    glusterfs2      Connected 
03075f3a-7e59-42a2-9285-40d241f572a1    localhost       Connected 

$> gluster peer status
Number of Peers: 1

Hostname: glusterfs2
Uuid: 3c510d0e-bc1d-4bf6-b41b-5bd4632b0d8a
State: Peer in Cluster (Connected)

```

```
$> gluster volume status
Status of volume: test-volume
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick glusterfs1:/data/glusterfs/test       49152     0          Y       166  
Brick glusterfs2:/data/glusterfs/test       49152     0          Y       130  
Self-heal Daemon on localhost               N/A       N/A        Y       189  
Self-heal Daemon on glusterfs2              N/A       N/A        Y       153  
 
Task Status of Volume test-volume
------------------------------------------------------------------------------
There are no active volume tasks
 
$> gluster volume list
test-volume

$> gluster volume info test-volume
 
Volume Name: test-volume
Type: Replicate
Volume ID: 2d9ae199-bb92-4fa4-bb40-5cbbf329f5bd
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: glusterfs1:/data/glusterfs/test
Brick2: glusterfs2:/data/glusterfs/test
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off


```

# Trouble Shooting

## Permission

Refz Doc:
 
* [Permission ACL](https://gluster.readthedocs.io/en/latest/Administrator%20Guide/Access%20Control%20Lists/)


1.Check logs under `/var/log/glusterfs/bricks`

```
$> ls
data-glusterfs-test.log
```

```
$> cat data-glusterfs-test.log

[2019-02-20 13:50:25.453787] I [MSGID: 139001] [posix-acl.c:269:posix_acl_log_permit_denied] 0-test-volume-access-control: 
client: 8bb22e062d10-90-2019/02/20-13:50:25:270692-test-volume-client-0-0-0, 
gfid: 00000000-0000-0000-0000-000000000001, 
req(uid:1000,gid:1000,perm:3,ngrps:0), 
ctx(uid:0,gid:0,in-groups:0,perm:755,updated-fop:LOOKUP, acl:-) [Permission denied]
```

2.Anaysis

Requested : `uid:1000,gid:1000,perm:3`
Asked : `uid:0,gid:0,in-groups:0,perm:755`

```
$> sudo getfacl /data/glusterfs/test/
getfacl: Removing leading '/' from absolute path names
# file: data/glusterfs/test/
# owner: root
# group: root
user::rwx
group::r-x
other::r-x
```

3.We need to grant the rw perm to uid:1000,gid:1000 

```
$> sudo setfacl -m u:1000:rwx /data/glusterfs/test
$> sudo setfacl -m u:1000:rwx /data/glusterfs/test

```