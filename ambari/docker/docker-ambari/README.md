# Ambari on Docker

This projects aim is to help you to get started with ambari. The 2 easiest way
to have an ambari server:

## Starting the container

This will start (and download if you never used it before) an image based on
centos-7 with preinstalled Ambari 2.1.0 ready to install HDP 2.3.

```
docker run -d -P -h ambari.esse.io --name ambari-server ambari
```

The explanation of the parameters:

- **-d**: run as daemon
- **-P**: expose all ports defined in the Dockerfile
- **-h ambari.esse.io**: sets the hostname
- **--name ambari-server**: sets the container name to **ambari-server** (no need to use )

## Cluster deployment via blueprint

Once the container is running, you can deploy a cluster. Instead of going to
the webui, we can use ambari-shell, which can interact with ambari via cli,
or perform automated provisioning. We will use the automated way, and of
course there is a docker image, with prepared ambari-shell in it:

```
docker run -e BLUEPRINT=single-node-hdfs-yarn --link amb0:ambariserver -t --rm --entrypoint /bin/sh sequenceiq/ambari-shell -c /tmp/install-cluster.sh
```

Ambari-shell uses Ambari's new [Blueprints](https://cwiki.apache.org/confluence/display/AMBARI/Blueprints)
capability. It just simple posts a cluster definition JSON to the ambari REST api,
and 1 more json for cluster creation, where you specify which hosts go
to which hostgroup.

Ambari shell will show the progress in the upper right corner.
So grab a cup coffee, and after about 10 minutes, you have a ready HDP 2.3 cluster.

## Multi-node Hadoop cluster

For the multi node Hadoop cluster instructions please read our [blog](http://blog.sequenceiq.com/blog/2014/06/19/multinode-hadoop-cluster-on-docker/) entry or run this one-liner:

```
curl -Lo .amb j.mp/docker-ambari && . .amb && amb-deploy-cluster
```

