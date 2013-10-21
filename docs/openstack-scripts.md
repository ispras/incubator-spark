---
layout: global
title: Running Spark on Openstack
---

The `spark_openstack.py` Python script located in Spark's `openstack` directory, allows you
to launch, manage and shut down Spark clusters on Openstack clusters. It automatically
sets up Spark and HDFS on the cluster for you.
This guide describes how to use `spark_openstack.py` to launch clusters, how to run jobs on
them, and how to shut them down.
It assumes the following:

-   You have access to Openstack dashboard (horizon) on your cluster
-   You have at least one instance flavor (type of instance) that has "Ephemeral disk"
-   You use the preconfigured one of system images of CentOS 6.4 from Modis
    team from Institute for System Programming of Russian Academy of Sciences.
    If your company policy doesn't allow you to use third-party system images
    or you are paranoid about data-safety, you can make an image yourself.
    Links to the list of system images and the instruction how to make your own
    image are at the bottom of this page.

`spark_openstack.py` is designed to manage multiple named clusters just as
`spark-ec2` does for Amazon. You can
launch a new cluster (telling the script its size and giving it a name),
shutdown an existing cluster, or log into a cluster. Each cluster is
identified by placing its machines into Openstack security groups whose names
are derived from the name of the cluster. For example, a cluster named
`test` will contain a master node in a security group called
`test-master`, and a number of slave nodes in a security group called
`test-slaves`. The `spark_openstack` script will create these security groups
for you based on the cluster name you request. You can also use them to
identify machines belonging to each cluster in the Openstack Horison 
web-interface.

`spark_openstack.py` is almost an exact clone of `spark-ec2` script for
Openstack platform. I uses native Openstack library `novaclient` for 
communications with Openstack. Nethertheless Openstack is more complicated to use
because it may have dozens of configurations, so you need to fullfill some of them
yourself (e.g. flavors and networks).


# Before You Start

-   Download special system image ([for spark-0.7.3 and spark-0.8.0 for now](http://spark.at.ispras.ru/)) or create your own one.
-   Upload it to your Openstack Cloud and save the identifier of uploaded image 
    (e.g. `7c2db475-4c33-477b-9f4a-821fc4265c6c`). You can do it in `Horizon/Images & Snapshots`
-   Download your Openstack RC file. You can do it on `Horizon/Access & Security` page in tab named `API access`
-   Run the script file you downloaded like this `. ./openstack-openrc.sh`. Pay attention to the first dot in the command!
-   Generate keypair and download it on `Horizon/Access & Security` page on tab named `Keypairs`. As a result you will download private key that will be used for ssh in future. Also remember the name you given to keypair.
-   Install python-novaclient library. You can do it via `pip install python-novaclient`.
-   Ask the administrator of your Openstack cluster to create special flavor (type of instance) with attached ephemeral disk (at least one is needed) with at least 7GB RAM.
-   Edit file `openstack/flavors.py`: you need to put the name given to your special flavor and put number of ephemeral disks given to the flavor after colon (e.g: `"o1.large": 1`)
-   Edit file `openstack/networks.py` and put in `openstack_networks` list of networks you want to attach to each instance. The first network should be internal to Openstack because instance gets initialization parameters using internal network. Also you should replace name of `floating_ips_pool_name` with the name of pool with external addresses used in your Openstack (e.g. `'ext'` => `'external network company_name'`). Maybe it should be better to ask your Openstack administrator for help.

# Launching a Cluster

-   Go into the `openstack` directory in the release of Spark you downloaded.
-   Run
    `python ./spark_openstack.py -a <Image ID to launch in Openstack> -k <keypair name> -i <ssh private key> -s <num-slaves> launch <cluster-name>`,
    
where `image ID` is ID of operating system image you want to use, `<keypair name>` is the name of your Openstack keypair (that you gave it when you created it), `<ssh private key>` is the private key file for your key pair, `<num-slaves>` is the number of slave nodes to launch (try
    1 at first), and `<cluster-name>` is the name to give to your
    cluster.
-   After everything launches, check that the cluster scheduler is up and sees
    all the slaves by going to its web UI, which will be printed at the end of
    the script (typically `http://<master-hostname>:8080`).

You can also run `python ./spark_openstack.py --help` to see more usage options. The
following options are worth pointing out:

-   `--instance-type=<INSTANCE_TYPE>` must be used to specify instance flavor. By default Openstack installations don't have flavors with Ephemeral storage but the scripts initializing the instance use Emphemeral storage for Ephemeral HDFS and some other nice things.
-   `--openstack-network-communication-method=<NET>` is intended to define how your instances will get IP addreses. For now there are two options: static addresses and floating addresses. Usually in production deployments floating addresses are used: in this case you will get external access only for your master instance, slave instances will be in separate internal network. It's needed because of floating addresses allocation quotas in Openstack. The second one (static) may be useful if you can see the network of your Openstack directly: e.g if you are testing something using [Devstack](http://devstack.org/)
-    If one of your launches fails due to e.g. not having the right
permissions on your private key file, you can run `launch` with the
`--resume` option to restart the setup process on an existing cluster.

# Running Jobs

-   Go into the `openstack` directory in the release of Spark you downloaded.
-   Run `python ./spark_openstack.py -k <keypair> -i <key-file> login <cluster-name>` to
    SSH into the cluster, where `<keypair>` and `<key-file>` are as
    above. (This is just for convenience; you could also Openstack Horizon or usual ssh-client
    for the address of your master instance.)
-   To deploy code or data within your cluster, you can log in and use the
    provided script `~/spark-openstack/copy-dir`, which,
    given a directory path, RSYNCs it to the same location on all the slaves.
-   If your job needs to access large datasets, the fastest way to do
    that is to load them from Amazon S3 or an Amazon EBS device into an
    instance of the Hadoop Distributed File System (HDFS) on your nodes.
    The `spark_openstack.py` script already sets up a HDFS instance for you. It's
    installed in `/root/ephemeral-hdfs`, and can be accessed using the
    `bin/hadoop` script in that directory. Note that the data in this
    HDFS goes away when you stop and restart a machine.
-   There is also a *persistent HDFS* instance in
    `/root/presistent-hdfs` that will keep data across cluster restarts.
    Typically each node has relatively little space of persistent data
    (about 3 GB)
-   You can view the status of the cluster using the Spark web UI
    (`http://<master-hostname>:8080`) or Ganglia Web Monitor for 
    cluster state and load (`http://<master-hostname>:5080/ganglia`) .

# Configuration

You can edit `/root/spark/conf/spark-env.sh` on each machine to set Spark configuration options, such
as JVM options. This file needs to be copied to **every machine** to reflect the change. The easiest way to
do this is to use a script we provide called `copy-dir`. First edit your `spark-env.sh` file on the master,
then run `~/spark-openstack/copy-dir /root/spark/conf` to RSYNC it to all the workers.

The [configuration guide](configuration.html) describes the available configuration options.

# Terminating a Cluster

***Note that there is no way to recover data on Openstack nodes after shutting
them down! Make sure you have copied everything important off the nodes
before stopping them.***

-   Go into the `openstack` directory in the release of Spark you downloaded.
-   Run `python ./spark_opentack destroy <cluster-name>`.

# Pausing and Restarting Clusters

The `spark_openstack.py` script also supports pausing a cluster. In this case,
the VMs are stopped but not terminated, so they
***lose all data on ephemeral disks*** but keep the data in their
root partitions and their `persistent-hdfs`. Stopped machines will not
cost you any processor time if you use commercial Openstack services
cycles.

- To stop one of your clusters, go into the `openstack` directory and run
`python ./spark_openstack stop <cluster-name>`.
- To restart it later, run
`python ./spark_openstack -i <key-file> start <cluster-name>`.
- To ultimately destroy the cluster and stop consuming EBS space, run
`python ./spark_openstack destroy <cluster-name>` as described in the previous
section.

# Limitations

- `spark_openstack.py` can launch instances only in one nova-zone.
- Script *MUST* be launched from the directory `<spark release>/openstack`. If you will launch it from another place, cluster creation process wil get stuck, because script will not be able to upload configuration templates for spark, hdfs etc.
- Script is untested in Windows (but there is no reasons to think that it will fail), try it on your own risk
- Remember that you need to upload special system image to your Openstack installation
- You need to edit file 'flavors.py' to contain the the name of your instance and the count of ephemeral disks it has
- You need flavor to contain ephemeral disk
- Cinder (analogue of ebs-storage in a sense) is not supported at the moment (*but it will be*)
- Links to spark workers in Spark web ui are broken, it Openstack-related issue.
- If you use Quantum (Neutron) Openstack component for network management you may face problems in some cases. In some configurations you may not be able to create security groups via `spark_openstack.py`. To resolve this issue you should comment out the following line in every `/etc/nova/nova.conf` on all Openstack compute nodes: `security_group_api =  quantum` =>  `# security_group_api =  quantum`
- We haven't check if system image works not under KVM - we just don't have another cluster

If you have a patch or suggestion for one of these limitations, feel free to
[contribute](contributing-to-spark.html) it!

# Using a Newer Spark Version

You always can make your own image or download the needed one from `http://spark.at.ispras.ru/`

# Creating your own Openstack-compatible system image for Spark

These instruction applies to KVM-based Openstack installations. All the actions are made using root account.

1. Install CentOS 6.4 using libvirt in any convenient way
2. Install package `java-1.7.0-openjdk` (or any compatible Java JDK)
3. Setup your JAVA_HOME
4. Download Scala: `wget http://scala-lang.org/files/archive/scala-2.9.3.tgz`
5. Unpack it and setup your SCALA_HOME
6. Install package `cloud-init`. You can see our configuration of cloud.cfg [here](http://spark.at.ispras.ru/)
7. Install ant
8. Download Spark distribution and Hadoop distribution corresponding to that build
9. Extract Hadoop distribution twice: the first one into /root/ephemeral-hdfs , the second one into /root/persistent-hdfs/
10. Install `rsync` package
11. Turn off services for iptables and ip6tables (you won't need them because Openstack controlls network security): `chkconfig iptables off`, `chkconfig ip6tables off`
12. Delete all identifiers from `/etc/sysconfig/network-scripts/ifcfg-eth0` and copy the contents to `ifcfg-eth1` (and more if you will have many networks in your Openstack)
13. Delete or comment out all the mentions about network 169.254* from `/etc/sysconfig/network-scripts/ifup-routes`. It's VERY important because Openstack uses special address 169.254.169.254 for instance initial configuration and CentOS routes this net to link-local.
14. (Optional) Replace `UseDNS yes` in your `/etc/ssh/sshd_config` with `UseDNS no`. If you don't have this record, then add `UseDNS no` to the end of file: this really speeds up cluster initialization
15. Shutdown your virtual machine
16. Install package that contains tool `virt-sysprep` in your host OS.
17. Run `sudo virt-sysprep --verbose -d <name of virtual machine you prepared>`
18. Upload the image if your Openstack-compatible image to Opentack -- everything's ready.
19. Follow the guide above
20. Make sure that everything works!