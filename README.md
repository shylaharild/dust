dust
====

DustCluster is an ssh cluster shell for EC2

Status:
* Tested/known to work on Linux only (Debian, Ubuntu, CentOS)
* Developed/tested with Python 2.7
* Currently, this is alpha/work in progress

[Installation and quick start](INSTALL.md)

## Rationale

DustCluster is an ssh cluster shell primarily useful for development, prototyping, one-off configuration of (usually ephemeral) EC2 clusters. 
Can be useful when developing custom data engineering stacks, maybe 10 nodes at most.

## Usage

### Ssh into and control existing EC2 clusters

Drop into a dust shell, and show all the nodes in the current region 

> dust$ show

```
Name         Instance     State        ID         ext_IP          int_IP         

             t2.nano      running      i-c8cbad4b 52.90.67.41     172.31.60.216  
             t2.nano      running      i-e6d4b265 52.90.136.102   172.31.61.174  
             t2.nano      running      i-1bd4b298 54.209.17.39    172.31.52.216  
             t2.nano      running      i-72caacf1 52.91.104.162   172.31.58.249  
             t2.nano      running      i-07d5b384 52.91.240.38    172.31.59.61
```

Or show with details:

> dust$ showex

```
Name         Instance     State        ID         ext_IP          int_IP         

             t2.nano      running      i-c8cbad4b 52.90.67.41     172.31.60.216  
              image : ami-8fcee4e5
              hostname : ec2-52-90-67-41.compute-1.amazonaws.com
              key : useast1_dustcluster
              DNS : ec2-52-90-67-41.compute-1.amazonaws.com
              tags : cluster=nano,name=node0

            .. etc..                   
```

Use filters to show/start/stop nodes:

> dust$ show state=running

> dust$ stop id=i-e6d4b265

Filter by tag:

> dust$ showex tags=name:node*

Select a cluster to work with:

> dust$ use filter tags=name:node*

This selects nodes with the tag name=node*, and saves down a template so that you can name nodes and address them 
by a friendly name (as you would in sshconfig).

Select all nodes again:

> dust$ use region us-east-1


### Start a new cluster

Optionally there is support to sync a very minimal cluster spec to the cloud. (You can write stateful commands to use
CloudFormation or Troposhpere for more elaborate specs.)

sample.yaml

```
cloud:
  provider: ec2 
  region: us-east-1

cluster:
  name: nano2

nodes:
- image: ami-60b6c60a
  instance_type: t2.nano
  nodename: worker1
  username: ec2-user
  key: ec2dust

- image: ami-60b6c60a
  instance_type: t2.nano
  nodename: worker2
  username: ec2-user
  key: ec2dust
```

> dust$ use template sample.yaml

> dust$ show

```
dust:dragonex$ show
dust:2014-09-14 08:29:22,234 | cluster 'democloud' in eu-west-1, using key: ec2dust
        Name     Instance        Image        State           ID           IP          DNS         tags 
Template Nodes:
     worker1     t2.small ami-892fe1fe  not_started                                                     
     worker0     t2.small ami-892fe1fe  not_started                                                     
     worker2     t2.small ami-892fe1fe  not_started                                                     
      master    m3.medium ami-892fe1fe  not_started                                                     
```


> dust$ start

> dust$ refresh

The nodes should be in the pending state, and the ID, IP and DNS fields populated.

**Note on authentication**:

Only key based authentication is supported. You can specify the key or keyfile in the template under each node.

### Target a set of nodes with wildcards and filter expressions

The basic node operations are start/stop/terminate 

with wildcards:

> dust$ stop worker\*

> dust$ start wo\*

> dust$ terminate worker[0-2]

with filter expressions:

> dust$ start state=stopped

> dust$ start state=stop*       # filters can have wildcards 

> dust$ stop tags=env:dev

The general form for node opertions is

> start/stop/terminate [target]

No target implies all nodes.


### Cluster ssh to a set of nodes

Execute 'uptime' over ssh on a set of nodes with:

> dust$ @worker\* uptime

Execute 'uptime' over ssh on all nodes with tag env=dev

> dust$ @tags=env:dev uptime

The general form for ssh is:

> dust$ @[target] command


e.g.

> dust$ @worker\* tail -2 /etc/resolv.conf

```
[worker0] nameserver 172.31.0.2
[worker0] search eu-west-1.compute.internal
[worker0] 


[worker1] nameserver 172.31.0.2
[worker1] search eu-west-1.compute.internal
[worker1] 


[worker2] nameserver 172.31.0.2
[worker2] search eu-west-1.compute.internal
[worker2] 
```

Again, [target] can have wildcards:

> dust$ @worker[0-2]  ls /var/log

> dust$ @w\*  ls /var/log

Or filter expressions:

> dust$ stop master

> dust$ @state=running  ls /var/log

> dust$ @id=i-c123*  ls /var/log

> dust$ @state=run\*  ls -l /var/log/auth.log

```
[worker0] -rw-r----- 1 syslog adm 398707 Sep  7 23:42 /var/log/auth.log
[worker0] 


[worker1] -rw-r----- 1 syslog adm 642 Sep  7 23:42 /var/log/auth.log
[worker1] 


[worker2] -rw-r----- 1 syslog adm 14470 Sep  7 23:46 /var/log/auth.log
[worker2] 
```

### These are demultiplexed fully interactive ssh shells !

So this works:

> dust$ @worker* cd /tmp

> dust$ @worker* pwd

```
[worker0] /tmp
[worker0] 

[worker1] /tmp
[worker1] 
```

And so does this:

> dust$ @worker0 sleep 10 && echo '10 second sleep done!' & 


> dust$ @worker0 ls -l /var/log/boot.log

```
[worker0] -rw------- 1 root root 0 Sep 19 13:18 /var/log/boot.log
[worker0] 

# 10 seconds later

[worker0] 10 second sleep done!
[worker0] 
```

And this:

> dust$ @worker\* sudo apt-get install nginx

```
.
.
[worker0] After this operation, 5,342 kB of additional disk space will be used.
[worker0] Do you want to continue? [Y/n] 

.
.
[worker1] After this operation, 5,342 kB of additional disk space will be used.
[worker1] Do you want to continue? [Y/n] 

.
.
[worker2] After this operation, 5,342 kB of additional disk space will be used.
[worker2] Do you want to continue? [Y/n] 
```

> dust$ @work\* Y

sends a Y to all the nodes named work\* and the apt-get script continues.

### Run vim or top on a single node, with the same ssh session ! 

> dust$ @worker2

The ssh '@' operator with a target and no commands enters a regular interactive ssh shell on a single node -- for running full screen console apps such as vim or top. 

This re-uses the same ssh session as the one above, but in char buffered mode:

> dust$ @worker* cd /tmp

> dust$ @worker0

> rawshell:worker0:/tmp$ pwd

```
/tmp
```


When done, log out of the ssh shell ($exit) or keep it going in the background (Ctrl-C x3) for future line 
buffered commands or raw shell mode.


### Add custom functionality with stateful drop-in python commands 

To add functionality, drop in a python file implementing new commands into dustcluster/commands. 

Out of the box commands: get (cluster download), put (cluster upload), setting up security groups, etc.

Type help or ? inside the dust shell for more

Unrecognized commands drop to the system shell, so you can edit files, run configuration management tools locally 
from the same prompt.

