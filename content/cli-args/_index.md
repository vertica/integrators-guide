---
title: "Command line arguments"
linkTitle: "Command line arguments"
weight: 4
---

The following files accept command line arguments:

- vertica
- bootstrap-catalog
- extract-snapshot
- vertica-download-file

## vertica Commands

The following command starts an existing database:

```bash
$ vertica -D /home/dbadmin/test/v_test_node0001_catalog -C test -n v_test_node0001 -h 127.0.0.1 -p 5433 -P 4803 -Y ipv4
```

Arguments that must be entered directly following the file name:

```
--version    (-V)    display version information, then exit  
--help       (-?)    show this help, then exit  
--stop       (-k)    stop the running process  
--status     (-s)    return status of the called process: 0 for running, 1 for non-existing  
--spread             doSpread = true
```
Arguments accepted anywhere in the command:

```
--force              make database consistent (clean up bad files and recover) 
--dualstackclient    dualStackClient = true`
```

### Flags (No arguments)

Listed in order of most commonly used to less commonly used:

```
-c    make database consistent (clean up bad files and recover) 
-I    run offline checksort index tool to check for correct projection sort order 
-T    disable tuple mover 
-v    run offline checkcrc index tool to check datafile integrity 
-4    accept incoming IPv4 connections from database clients 
-6    accept incoming IPv6 connections from database clients 
-B    standby = true 
-e    encryptSpread = true 
-f    useForce = true, regularStart = false 
-s    doStatus = true, regularStart = false 
-U    gRecover.setRecoveryDisabled(true) 
-z    zippyCatalogRead = true, catLoadParams.zippyCatalogRead = true
```
### Flags (Accepts arguments)

```
-C <NAME>     database name 
-D <DIR>      database directory (required)
-h <HOST>     hostname 
-n <NODE>     nodename 
-p <PORT>     port number to listen on 
-l <LEVEL>    logging level 
-L <LFILE>    validate license file then exit 
-m <MPATH>    measure location performance then exit 
-P <PORT>     spread port number 
-S <EPOCH>    recovery epoch number 
-Y <IPV4|6>   IP address family for Spread and intracluster communication 
-E <ARG>      editorCmds = arg or interactive, regularStart = false 
-I <ARG>      verifySortOrder = true, indextoolThreads = arg, regularStart = false, regularStart = false 
-o <ARG>      output = arg 
-t <FILE>     triggerFile = arg 
-v <ARG>      validateDatabaseThoroughly = true, indextoolThreads = arg, regularStart = false`
```

## bootstrap-catalog

Use the bootstrap-catalog command to create the catalog on a new node, or to restore from a snapshot. For example, to create a new database, run the following command:

```bash
$ bootstrap-catalog -C test \ 
   -H 127.0.0.1 \ 
   -s v_test_node0001 \ 
   -D /home/dbadmin/test/v_test_node0001_catalog \ 
   -S /home/dbadmin/test/v_test_node0001_data \ 
   -p 5433 \ 
   -c 127.0.0.1 \ 
   -B 127.255.255.255 \ 
   -L /opt/vertica/config/share/license.key \ 
   -x 4803 \ 
   -A password \ 
   -a <password>
```

### Flags

```bash
--help         (-h)    Displays the below options`
```

If you are initializing a new catalog, the following options are required:

```bash
--catalogpath  (-D)    <pathname>  
--databasename (-C)    <databasename>  
--nodename     (-n)    <nodename>  
--sitename     (-s)    <nodename> (deprecated usage)
```

If you are restoring from a snapshot, the following options are required:

```bash
--catalogpath  (-D)    <pathname>  
--filename     (-F)    <filename.ctlg>
```

If restoring from a udstorage snapshot, the extra option is
required:

```bash
--fssnapshot   (-u)    <filename.udfs>
```

Other options:

```
--license              (-L)    <keypath> Supplies the license file to install 
--password             (-a)    Supplies the password for password auth method 
--auth                 (-A)    Specifies the authentication method for this database: "trust" or "password", defaults to trust 
--host                 (-H)    Specifies the hostname/address for this node 
--broadcast            (-B)    Specifies the broadcast address for this node 
--port                 (-p)    Specifies the client port for this node 
--overwrite            (-O)    Overwrite any existing Catalog 
--pwprompt             (-P)    Prompt for password, if none is supplied 
--storage              (-S)    Specifies the Unix path to be used for data storage 
--configparam          (-X)    <configparam>=<value>   Sets param to value in vertica.conf (can be repeated) 
--nodecount            (-N)    <#nodes> Hint to how many nodes will be in cluster 
--controladdr          (-c)    <address> Specifies the ip address to use for spread
--controladdrfamily    (-Y)    Specifies the IP address family for Spread; one of 'ipv4','ipv6' 
--spreadport           (-x)    Specifies the port to use for spread 
--nobroadcast          (-T)    Specifies not to use UDP broadcast for spread 
--spreadlogging        (-l)    Enable spread logging to catalogpath/../spread.log 
--copycluster          (-z)    Specifies cross-cluster restore or copy operation 
--segcount             (-K)    <#segshards> Used in EON mode. Specifies node parallelism for shared storage deployment 
--sharedstorage        (-G)    <catTruncateVer:catTargetVer> Used in EON mode. Create shared storage location. Overloaded to issue catalog truncation to a prior catalog version 
--load-remote-catalog  (-R)    Loads a catalog from an existing remote shared location 
--hosts                (-I)    Specifies the IP addresses for all the hosts in the cluster 
--check-directory      (-d)    Check the catalog directory for this node`
```

## extract-snapshot Commands

```
Usage: extract-snapshot -d DB_DIR -s SNAP_NAME -S RESTORE_SNAP_NAME -t
{full|object} -o OBJECTS [flags]
```

### Flags

```
-c                        run cluster update tasks and generate catalog diff 
-C                        apply catalog diff 
-d <DB_DIR>               database directory 
-o <OBJECTS>              database directory (required) 
-i <ARG>                  incObjectStrs = ARG 
-e <ARG>                  excObjectStrs = ARG 
-S <RESTORE_SNAP_NAME>    name of snapshot to restore 
-s <SNAP_NAME>            snapshot name 
-t <full|object>          is the object a backup (full means no)`
```

## vertica-download-file Commands

```
Usage: vertica-download-file -a sourceFile -b destinationFile -1
logDirectory
```

### Flags

```
--source-file.        (-a)    <sourceFile>  
--destination-file    (-b)    <destinationFile>  
--logdir              (-1)    <logDirectory>  
--configparam         (-X)    <configureParameters>  
--catalogpath         (-D)    <databasePath>
```

## Bring Up a Vertica Database

```
mkdir <directory>  
vertica-bootstrap-catalog --nodename <nodeName> -D <catalog path> -T <datapath> -H <hostname>  
vertica -D <catalog path>
```

The parameters nodeName, catalogpath, and hostname are mandatory and have same semantics as nodeName, CATALOGPATH, node address in CREATE NODE.