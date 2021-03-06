# Run Your Own Miner

![](../.gitbook/assets/artboard-copy-57%20%281%29.jpg)

Running a Helium Miner is a great way to get some exposure to the blockchain and to support the network. If you have [your own hardware deployed](../hotspot/developer-setup.md), this is necessary for routing LoRaWAN packets according to our [LongFi](https://developer.helium.com/longfi/introduction) protocol.

![](../.gitbook/assets/artboard%20%281%29.png)

As you can see above, the Miner is central in routing data across the Helium Network. It is one of three pieces:

* Packet Forwarder: this is a utility that interacts with the radio front-end and sends and receives raw radio packets with the Helium Miner
* Miner: the Helium Blockchain comes into the picture here; the Miner is responsible for routing packets to the appropriate Router \(see [our Routing article](../longfi/longfi-routing.md)\) and entering into micro-transactions brokered via libp2p
* Router: a Helium compatible LoRaWAN Network Server, basically; this component is interested in receiving the packets relating to its devices and handles downlink messages when appropriate

In addition to packet routing, the Miner is central in connecting to other Miners over libp2p where, amongst other things, it is gossiping and saving blocks, while maintaining a ledger of the blockchain.

In this guide, we will explain how to get a Docker image of the Helium Miner running on Ubuntu 20.04 LTS, and finally some tips on how to interact with a running Miner.

If you are interested in contributing to Miner development and code-review, please visit [the Miner's repository on Github](https://github.com/helium/miner).

## Setting Up Ubuntu for Docker

Ubuntu is a widely available Linux distribution. Notably, it has an easy-to-use image available for Raspberry Pi 3 and 4, so we use it as an example system. That being said, any ARM64 or AMD64 \(X86-64\) based OS that can run Docker is suitable.

For Raspberry Pi, [select the 64-bit version of Ubuntu](https://ubuntu.com/download/raspberry-pi) for your appropriate model. We currently do not have Docker image support for 32-bit systems, so please double-check that you're using a 64-bit image. Once you have followed the instructions and are logged into the system, you are ready for the rest of this guide.

For most cloud service providers, launching an instance with Ubuntu 20.04 LTS should be fairly straightforward. With AWS for example, create an EC2 instance running Ubuntu 20.04. A t2.small will run the miner well once the initial blockchain sync has completed. Once that's launched and you're connected, you are ready for the rest of this guide.

First, update the package manager registry:

```text
sudo apt-get update
```

Then, install Docker:

```text
sudo apt-get install docker.io
```

To avoid needing to user docker with `sudo` privileges, add your user to the `docker` group, replacing $USER with your username:

```text
sudo usermod -aG docker $USER
```

Log in and out of your account to apply these changes. You are now ready to use Docker!

## Port Forwarding

Before launching the Miner, you will want to configure ports on your network to forward two ports:

* **44158/TCP**: the Miner communicates to other Miners over this port. The networking logic knows how to get around a lack of forwarding here, but you will get better performance by forwarding the port
* **1680/UDP**: the radio connects to the Miner over this port. You will not be able to forward packets or participate in Proof of Coverage without this

"Forwarding" on the second port is less relevant if you are running a radio packet forwarder on the same system at the miner \(both on a Raspberry Pi, for example\). But it is essential if you are running a Miner on the cloud for example.

For AWS, for example, you will want to configure the "Security Group" of your EC2 as so:

![](../.gitbook/assets/security-group%20%281%29.png)

## Run a Docker Container

Miner releases are available as amd64 and arm64 images on at [quay.io](https://quay.io/repository/team-helium/miner?tab=tags). We do not currently provide 32-bit support.

### Start Container

Before running the container for the first time, it is advisable to pick a 'system mount point\`, ie: a directory in your host system; some long-term miner data is stored there. This allows you to easily maintain your miner's blockchain identity \(ie: swarm keys\) and blockchain state through miner image updates.

If you are using a Linux system, you can just create a directory in your user's home directory:

```text
mkdir ~/miner_data
```

If you are using Ubuntu as user ubuntu, this path would now be `/home/ubuntu/miner_data`. This will be used later.

Now you can try the `run` command to start your container for the first time:

```text
docker run -d \
--env REGION_OVERRIDE=US915 \
--restart always \
--publish 1680:1680/udp \
--publish 44158:44158/tcp \
--name miner \
--mount type=bind,source=/home/ubuntu/miner_data,target=/var/data \
quay.io/team-helium/miner:miner-xxxNN_YYYY.MM.DD
```

Replace xxxNN with the architecture used, ie: amd64 or arm64, and with the release date of the image.

The `-d` option runs in detached mode, which makes the command return or not; you may want to omit if you have a daemon manager running the docker for you.

The `-env REGION_OVERRIDE=US915` tells your miner that you are connecting to a packet forwarder configured for the US915 region. You will want to change this to your region, ie:  
`US915 | EU868 | EU433 | CN470 | CN779 | AU915 | AS923 | KR920 | IN865`

> Note: REGION\_OVERRIDE may be completely omitted once your Miner has asserted location and is fully synced, but leaving it there \(as long as the region is properly configured\) is not harmful

The `--restart always` option asks Docker to keep the image running, starting the image on boot and restarting the image if it crashes. Depending on how you installed Docker in your system, it'll start on boot. In the AWS AMI above, we use systemd \(`systemctl status docker` to check\).

The `--publish 1680:1680/udp` binds your system port 1680 to the containers port 1680, where the Miner is hosting a packet forwarder UDP server; this is necessary if you want to do any radio interactions with your miner.

The `--name miner` names the container, which makes interacting with the docker easier, but feel free to name the container whatever you want.

The `--mount` with the parameters above will mount the container's `/var/data/` directory to the systems directory `/home/ec2-user/miner_data`.

### Configure AWS Instance for Sync

Amazon EC2 instances have CPU usage credits that will easily be depleted during the initial sync of the blockchain. To avoid having your instance throttled, you can temporarily uncap your instance by setting CPU credit usage to unlimited. Once your miner has reached full block height, a t2.small instance is sufficient to keep your miner running.

### Interact with the Miner within the Container

You may want to interrogate the Miner or interact with it it as described in [Using the Miner](run-your-own-miner.md#using-the-miner). Docker's `exec` command enables this, eg:

```text
docker exec miner miner info height
```

In other words, prepend `docker exec miner` to any of the commands documented in Using [Using the Miner](run-your-own-miner.md#using-the-miner). Or create an alias such as: `alias miner="docker exec miner miner"`

### Updating the Docker Image

From time to time, the Helium Miner is updated. Keep tabs on [the releases here](https://github.com/helium/miner/releases). Depending on whether you are running a miner for fun, to route packets, or to participate in POC, keeping it updated may be more or less urgent. Each release tagged on the Github will be on the quay repository. Simply remove the current image:

```text
docker stop miner && docker rm miner
```

And [Start the Container ](run-your-own-miner.md#start-container)again as described above, but with the new release tag! Thanks to the `--mount` option, the blockchain data and the miner keys are preserved through updates.

## Using the Miner

These commands will assume you are running in Docker and they have the same prefix to get you exceuting a command within the docker: `docker exec miner` . If you want to make it easier, you can always created an an alias such as: `alias miner="docker exec miner miner"`.

### Checking the logs

This is always helpful to get some idea of what's going on:

```text
docker exec miner tail -F /var/log/miner/console.log
```

Also, if you are particularly interested in radio traffic, it can be helpful to filter for `lora` as so:

```text
docker exec miner tail -f /var/log/miner/console.log | grep lora
```

### Checking the peer-to-peer network

This is the first health check you can do to see how your Miner is doing. Is it finding other Helium miners over libp2p properly?

The Helium blockchain uses an Erlang implementation of [libp2p](https://libp2p.io/). Because we expect Hotspot hardware to be installed in a wide variety of networking environments [erlang-libp2p](https://github.com/helium/erlang-libp2p) includes a number of additions to the core specification that provides support for NAT detection, proxying and relaying.

The first order of business once the Miner is running is to see if you're connected to any peers, whether your NAT type has been correctly identified, and that you have some listen addresses:

```text
docker exec miner miner peer book -s
```

You will see an output roughly like the following:

```bash
+--------------------------------------------------------+------------------------------+------------+-----------+---------+------------+
|                        address                         |             name             |listen_addrs|connections|   nat   |last_updated|
+--------------------------------------------------------+------------------------------+------------+-----------+---------+------------+
|/p2p/11dwT67atkEe1Ru6xhDqPhSXKXmNhWf3ZHxX5S4SXhcdmhw3Y1t|{ok,"genuine-steel-crocodile"}|     2      |     4     |symmetric|   3.148s   |
+--------------------------------------------------------+------------------------------+------------+-----------+---------+------------+

+----------------------------------------------------------------------------------------------------------------------------+
|                                                 listen_addrs (prioritized)                                                 |
+----------------------------------------------------------------------------------------------------------------------------+
|/p2p/11apmNb8phR7WXMx8Pm65ycjVY16rjWw3PvhSeMFkviWAUu9KRD/p2p-circuit/p2p/11dwT67atkEe1Ru6xhDqPhSXKXmNhWf3ZHxX5S4SXhcdmhw3Y1t|
|                                                 /ip4/192.168.3.6/tcp/36397                                                 |
+----------------------------------------------------------------------------------------------------------------------------+

+--------------------------+-----------------------------+---------------------------------------------------------+------------------------------+
|          local           |           remote            |                           p2p                           |             name             |
+--------------------------+-----------------------------+---------------------------------------------------------+------------------------------+
|/ip4/192.168.3.6/tcp/36397|/ip4/104.248.122.141/tcp/2154|/p2p/112GZJvJ4yUc7wubREyBHZ4BVYkWxQdY849LC2GGmwAnv73i5Ufy|{ok,"atomic-parchment-snail"} |
|/ip4/192.168.3.6/tcp/36397| /ip4/73.15.36.201/tcp/13984 |/p2p/112MtP4Um2UXo8FtDHeme1U5A91M6Jj3TZ3i2XTJ9vNUMawqoPVW|   {ok,"fancy-glossy-rat"}    |
|/ip4/192.168.3.6/tcp/36397| /ip4/24.5.52.135/tcp/41103  |/p2p/11AUHAqBatgrs2v6j3j75UQ73NyEYZoH41CdJ56P1SzeqqYjZ4o |  {ok,"skinny-fuchsia-mink"}  |
|/ip4/192.168.3.6/tcp/46059| /ip4/34.222.64.221/tcp/2154 |/p2p/11LBadhdCmwHFnTzCTucn6aSPieDajw4ri3kpgAoikgnEA62Dc6 | {ok,"skinny-lilac-mustang"}  |
+--------------------------+-----------------------------+---------------------------------------------------------+------------------------------+
```

As long as you have an address listed in `listen_addrs` and some peers in the table at the bottom, you're connected to the p2p network and good to go.

If you are having trouble, try forwarding port `44158` to the miner. On AWS, double check your security group settings.

### Checking Block Height

As long as a genesis block is loaded, this query will work and return height 1 at least:

```text
docker exec miner miner info height
```

If you are syncing, you should see something like this:

```text
~$ miner info height
6889        280756
```

The first number is the election epoch and the second number is the block height of your miner. If you just launched an AMI instance, your Miner is been disconnected, or you simply have a slow connection, you may be a few blocks behind. To check, you can either check the mobile app, check [the browser-based block explorer](https://network.helium.com/blocks), or run a simple curl command to check in a Terminal:

```text
~$ curl https://api.helium.io/v1/blocks/height
{"data":{"height":280756}}
```

### Backing Up Your Swarm Keys

Periodically, we may release updates or you may want to migrate your miner from one machine to another. The most important thing is that you maintain a copy of your miner's private key, ie: `swarm_key`. Fun tidbit: this is also how your three-word name is generated.

Assuming you've mounted the Docker image as detailed above, it is located at:

```text
~/miner_data/miner/swarm_key
```

Another fun tidbit: for production hotspots sold by Helium, the swarm key is stored inside of a secure element and is thus unable to be migrated \(or compromised/accidentally lost unless physically damaged\).

### Providing Coverage

While participating in libp2p is helpful for the network, the Helium Blockchain does not exist for its own sake. It is there to incentivize coverage and one of the ways to earn tokens as a coverage provider for Helium is by routing IoT traffic.

To learn more about this, check out the [Build a Hotspot](../hotspot/developer-setup.md) section.

