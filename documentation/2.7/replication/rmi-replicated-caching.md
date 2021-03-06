---
---
# Replicated Caching using RMI


## Introduction

Replicated caching using RMI is desirable because:

* RMI is the default remoting mechanism in Java
* it allows tuning of TCP socket options
* Element keys and values for disk storage must already be Serializable, therefore directly transmittable over RMI
    without the need for conversion to a third format such as XML.
* it can be configured to pass through firewalls

![Ehcache Image](/images/documentation/rmi_replication.png)

While RMI is a point-to-point protocol, which can generate a lot of network
traffic, Ehcache manages this through batching of communications for the
asynchronous replicator.

To set up replicated caching with RMI you need to configure the CacheManager with:

* a PeerProvider
* a CacheManagerPeerListener

For each cache that will be replicated, you then need to add
one of the RMI cacheEventListener types to propagate messages.
You can also optionally configure a cache to bootstrap from other caches in the cluster.

## Suitable Element Types

Only Serializable Elements are suitable for replication.

Some operations, such as remove, work off Element keys rather than the full Element itself. In this case
the operation will be replicated provided the key is Serializable, even if the Element is not.

## Configuring the Peer Provider

### Peer Discovery

Ehcache has the notion of a group of caches acting as a replicated
cache. Each of the caches is a peer to the others. There is no
master cache. How do you know about the other caches that are in your
cluster? This problem can be given the name Peer Discovery.
Ehcache provides two mechanisms for peer discovery:
manual and automatic.

To use one of the built-in peer discovery mechanisms, specify the class
attribute of `cacheManagerPeerProviderFactory` as
`net.sf.ehcache.distribution.RMICacheManagerPeerProviderFactory`
in the ehcache.xml configuration file.

### Automatic Peer Discovery

Automatic discovery uses TCP multicast to establish and maintain a
multicast group. It features minimal configuration and automatic
addition to and deletion of members from
the group. No a priori knowledge of the servers in the
cluster is required. This is recommended as the default option.
Peers send heartbeats to the group once per second. If a peer has
not been heard of for 5 seconds it is dropped from the group. If a new
peer starts sending heartbeats it is admitted to the group.

Any cache within the configuration set up as replicated will be made
available for discovery by other peers.

To set automatic peer discovery, specify the properties attribute of `cacheManagerPeerProviderFactory`
as follows:

peerDiscovery=automatic  
multicastGroupAddress=multicast address | multicast host name  
multicastGroupPort=port  
timeToLive=0-255 _(See below in common problems before setting this)_  
hostName=_the hostname or IP of the interface to be used for sending and receiving multicast packets
      (relevant to mulithomed hosts only)_

#### Example
Suppose you have two servers in a cluster, server1 and server2. You wish to distribute
sampleCache11 and sampleCache12. The configuration
required for each server is identical, so the configuration for both server1 and server2 is the following:

    <cacheManagerPeerProviderFactory
    class="net.sf.ehcache.distribution.RMICacheManagerPeerProviderFactory"
    properties="peerDiscovery=automatic, multicastGroupAddress=230.0.0.1,
    multicastGroupPort=4446, timeToLive=32"/>

### Manual Peer Discovery <a name="Manual Peer Discovery"/>

Manual peer configuration requires the IP address and port of each
listener to be known. Peers cannot be added or removed at
runtime. Manual peer discovery is recommended where there are technical
difficulties using multicast, such as a router between servers in a
cluster that does not propagate multicast datagrams. You can also use
it to set up one way replications of data, by having server2 know about
server1 but not vice versa.

To set manual peer discovery, specify the properties attribute
of `cacheManagerPeerProviderFactory` as follows:

peerDiscovery=manual  
rmiUrls=//server:port/cacheName, ...  

The rmiUrls is a list of the cache peers of the server being
configured. Do not include the server being configured in the list.

#### Example

Suppose you have two servers in a cluster, server1 and server2. You wish to distribute
sampleCache11 and sampleCache12. The following is the configuration
required for server1:

    <cacheManagerPeerProviderFactory
    class="net.sf.ehcache.distribution.RMICacheManagerPeerProviderFactory"
    properties="peerDiscovery=manual,
    rmiUrls=//server2:40001/sampleCache11|//server2:40001/sampleCache12"/>

The following is the configuration required for server2:


    <cacheManagerPeerProviderFactory
    class="net.sf.ehcache.distribution.RMICacheManagerPeerProviderFactory"
    properties="peerDiscovery=manual,
    rmiUrls=//server1:40001/sampleCache11|//server1:40001/sampleCache12"/>


## Configuring the CacheManagerPeerListener

A CacheManagerPeerListener listens for messages from peers to the
current CacheManager.

You configure the CacheManagerPeerListener by specifiying a
CacheManagerPeerListenerFactory which is used to create the
CacheManagerPeerListener using the plugin mechanism.

The attributes of cacheManagerPeerListenerFactory are:

* class - a fully qualified factory class name
* properties - comma separated properties having meaning only to the factory.

Ehcache comes with a built-in RMI-based distribution system. The
listener component is RMICacheManagerPeerListener which is configured
using RMICacheManagerPeerListenerFactory. It is configured as per the
following example:


    <cacheManagerPeerListenerFactory
    class="net.sf.ehcache.distribution.RMICacheManagerPeerListenerFactory"
    properties="hostName=localhost, port=40001,
    socketTimeoutMillis=2000"/>


Valid properties are:

* hostName (optional) - the hostName of the host the listener is
running on. Specify where the host is multihomed and you want to
control the interface over which cluster messages are received.
The hostname is checked for reachability during CacheManager initialisation.
If the hostName is unreachable, the CacheManager will refuse to start and an
CacheException will be thrown indicating connection was refused.
If unspecified, the hostname will use `InetAddress.getLocalHost().getHostAddress()`, which corresponds to the default
host network interface.
Warning: Explicitly setting this to localhost refers to the local loopback of 127.0.0.1, which is not network visible
and will cause no replications to be received from remote hosts. You should only use this setting when multiple
CacheManagers are on the same machine.
* port (mandatory) - the port the listener listens on.
* socketTimeoutMillis (optional) - the number of seconds client sockets will wait
when sending messages to this listener until they give up. By default
this is 2000ms.

## Configuring Cache Replicators

Each cache that will be replicated needs to set a cache event listener
which then replicates messages to the other CacheManager peers. This is
done by adding a cacheEventListenerFactory element to each cache's
configuration.


    <!-- Sample cache named sampleCache2. -->
    <cache name="sampleCache2"
      maxEntriesLocalHeap="10"
      eternal="false"
      timeToIdleSeconds="100"
      timeToLiveSeconds="100"
      overflowToDisk="false">
    <cacheEventListenerFactory
    class="net.sf.ehcache.distribution.RMICacheReplicatorFactory"
    properties="replicateAsynchronously=true, replicatePuts=true, replicateUpdates=true,
    replicateUpdatesViaCopy=false, replicateRemovals=true "/>
    </cache>


class - use net.sf.ehcache.distribution.RMICacheReplicatorFactory

The factory recognises the following properties:

* replicatePuts=true | false - whether new elements placed in a cache are
 replicated to others. Defaults to true.
* replicateUpdates=true | false - whether new elements which override an
 element already existing with the same key are replicated. Defaults to true.
* replicateRemovals=true - whether element removals are replicated. Defaults to true.
* replicateAsynchronously=true | false - whether replications are
 asyncrhonous (true) or synchronous (false). Defaults to true.
* replicateUpdatesViaCopy=true | false - whether the new elements are
 copied to other caches (true), or whether a remove message is sent. Defaults to true.

To reduce typing if you want default behaviour, which is replicate everything in asynchronous mode, you can leave off
the `RMICacheReplicatorFactory` properties as per the following example:


    <!-- Sample cache named sampleCache4. All missing RMICacheReplicatorFactory properties
        default to true -->
    <cache name="sampleCache4"
          maxEntriesLocalHeap="10"
          eternal="true"
          overflowToDisk="false"
          memoryStoreEvictionPolicy="LFU">
       <cacheEventListenerFactory
           class="net.sf.ehcache.distribution.RMICacheReplicatorFactory"/>
    </cache>


## Configuring Bootstrap from a Cache Peer

 When a peer comes up, it will be incoherent with other caches. When the bootstrap completes it will be partially
 coherent. Bootstrap gets the list of keys from a random peer, and then loads those in batches from random
 peers. If bootstrap fails then the Cache will not start. However if a cache replication operation occurs
 which is then overwritten by bootstrap there is a chance that the cache could be inconsistent.

 Here are some scenarios:

**Delete overwritten by boostrap put**  
Cache A keys with values: 1, 2, 3, 4, 5  
Cache B starts bootstrap  
Cache A removes key 2  
Cache B removes key 2 and then bootstrap puts it back  

**Put overwritten by boostrap put**  
Cache A keys with values: 1, 2, 3, 4, 5  
Cache B starts bootstrap  
Cache A updates the value of key 2  
Cache B updates the value of key 2 and then bootstrap overwrites it with the
old value  


 The solution is for bootstrap to get a list of keys and write them all before committing transactions.

 This could cause synchronous transaction replicates to back up. To solve this problem, commits will be
  accepted, but not written to the cache until after bootstrap. Coherency is maintained because the cache
  is not available until bootstrap has completed and the transactions have been completed.

## Full Example

Ehcache's own integration tests provide complete examples of RMI-based replication.

  The best example is the integration test for cache replication. You can see it online here:
  [http://ehcache.org/xref-test/net/sf/ehcache/distribution/RMICacheReplicatorTest.html](http://ehcache.org/xref-test/net/sf/ehcache/distribution/RMICacheReplicatorTest.html)

The test uses five ehcache.xml files representing five CacheManagers set up to replicate using RMI.

## Common Problems

### Tomcat on Windows

 Any RMI listener will fail to start on Tomcat, if the installation path has spaces in it. Because the default on Windows is to install Tomcat in "Program Files", this issue will occur by default. The workaround is to remove the spaces in your Tomcat installation path.

### Multicast Blocking

 The automatic peer discovery process relies on multicast. Multicast can be blocked by routers.
 Virtualisation technologies like Xen and VMWare may be blocking multicast. If so enable it.
 You may also need to turn it on in the configuration for your network interface card.
 An easy way to tell if your multicast is getting through is to use the Ehcache remote debugger
 and watch for the heartbeat packets to arrive.

### Multicast Not Propagating Far Enough or Propagating Too Far

 You can control how far the multicast packets propagate by setting the badly misnamed time to live.
 Using the multicast IP protocol, the timeToLive value indicates the scope or range in which a packet may be forwarded.

By convention:

0 is restricted to the same host  
1 is restricted to the same subnet  
32 is restricted to the same site  
64 is restricted to the same region  
128 is restricted to the same continent  
255 is unrestricted

The default value in Java is 1, which propagates to the same subnet. Change the
timeToLive property to restrict or expand propagation.
