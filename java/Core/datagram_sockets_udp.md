# Datagram sockets (UDP)

Using User Datagram Protocol (UDP) with Vert.x is a piece of cake.

UDP is a connection-less transport which basically means you have no persistent connection to a remote peer.

Instead you can send and receive packages and the remote address is contained in each of them.

Beside this UDP is not as safe as TCP to use, which means there are no guarantees that a send Datagram packet will receive it’s endpoint at all.

The only guarantee is that it will either receive complete or not at all.

Also you usually can’t send data which is bigger then the MTU size of your network interface, this is because each packet will be send as one packet.

But be aware even if the packet size is smaller then the MTU it may still fail.

At which size it will fail depends on the Operating System etc. So rule of thumb is to try to send small packets.

Because of the nature of UDP it is best fit for Applications where you are allowed to drop packets (like for example a monitoring application).

The benefits are that it has a lot less overhead compared to TCP, which can be handled by the NetServer and NetClient (see above).

## Creating a DatagramSocket
To use UDP you first need t create a DatagramSocket. It does not matter here if you only want to send data or send and receive.
```
DatagramSocket socket = vertx.createDatagramSocket(new DatagramSocketOptions());
```
The returned DatagramSocket will not be bound to a specific port. This is not a problem if you only want to send data (like a client), but more on this in the next section.

## Sending Datagram packets
As mentioned before, User Datagram Protocol (UDP) sends data in packets to remote peers but is not connected to them in a persistent fashion.

This means each packet can be sent to a different remote peer.

Sending packets is as easy as shown here:
```
DatagramSocket socket = vertx.createDatagramSocket(new DatagramSocketOptions());
Buffer buffer = Buffer.buffer("content");
// Send a Buffer
socket.send(buffer, 1234, "10.0.0.1", asyncResult -> {
  System.out.println("Send succeeded? " + asyncResult.succeeded());
});
// Send a String
socket.send("A string used as content", 1234, "10.0.0.1", asyncResult -> {
  System.out.println("Send succeeded? " + asyncResult.succeeded());
});
```

## Receiving Datagram packets
If you want to receive packets you need to bind the DatagramSocket by calling listen(…​)} on it.

This way you will be able to receive DatagramPacket`s that were sent to the address and port on which the `DatagramSocket listens.

Beside this you also want to set a Handler which will be called for each received DatagramPacket.

The DatagramPacket has the following methods:

* sender: The InetSocketAddress which represent the sender of the packet

* data: The Buffer which holds the data which was received.

So to listen on a specific address and port you would do something like shown here:
```
DatagramSocket socket = vertx.createDatagramSocket(new DatagramSocketOptions());
socket.listen(1234, "0.0.0.0", asyncResult -> {
  if (asyncResult.succeeded()) {
    socket.handler(packet -> {
      // Do something with the packet
    });
  } else {
    System.out.println("Listen failed" + asyncResult.cause());
  }
});
```
Be aware that even if the {code AsyncResult} is successed it only means it might be written on the network stack, but gives no guarantee that it ever reached or will reach the remote peer at all.

If you need such a guarantee then you want to use TCP with some handshaking logic build on top.

## Multicast
### Sending Multicast packets

Multicast allows multiple sockets to receive the same packets. This works by have same join a multicast group to which you can send packets.

We will look at how you can joint a Multicast Group and so receive packets in the next section.

For now let us focus on how to send those. Sending multicast packets is not different to send normal Datagram Packets.

The only difference is that you would pass in a multicast group address to the send method.

This is show here:
```
DatagramSocket socket = vertx.createDatagramSocket(new DatagramSocketOptions());
Buffer buffer = Buffer.buffer("content");
// Send a Buffer to a multicast address
socket.send(buffer, 1234, "230.0.0.1", asyncResult -> {
  System.out.println("Send succeeded? " + asyncResult.succeeded());
});
```
All sockets that have joined the multicast group 230.0.0.1 will receive the packet.

#### Receiving Multicast packets

If you want to receive packets for specific Multicast group you need to bind the DatagramSocket by calling listen(…​) on it and join the Multicast group.

This way you will be able to receive DatagramPackets that were sent to the address and port on which the DatagramSocket listens and also to those sent to the Multicast group.

Beside this you also want to set a Handler which will be called for each received DatagramPacket.

The DatagramPacket has the following methods:

* sender(): The InetSocketAddress which represent the sender of the packet

* data(): The Buffer which holds the data which was received.

So to listen on a specific address and port and also receive packets for the Multicast group 230.0.0.1 you would do something like shown here:
```
DatagramSocket socket = vertx.createDatagramSocket(new DatagramSocketOptions());
socket.listen(1234, "0.0.0.0", asyncResult -> {
  if (asyncResult.succeeded()) {
    socket.handler(packet -> {
      // Do something with the packet
    });

    // join the multicast group
    socket.listenMulticastGroup("230.0.0.1", asyncResult2 -> {
        System.out.println("Listen succeeded? " + asyncResult2.succeeded());
    });
  } else {
    System.out.println("Listen failed" + asyncResult.cause());
  }
});
```

#### Unlisten / leave a Multicast group

There are sometimes situations where you want to receive packets for a Multicast group for a limited time.

In this situations you can first start to listen for them and then later unlisten.

This is shown here:
```
DatagramSocket socket = vertx.createDatagramSocket(new DatagramSocketOptions());
socket.listen(1234, "0.0.0.0", asyncResult -> {
    if (asyncResult.succeeded()) {
      socket.handler(packet -> {
        // Do something with the packet
      });

      // join the multicast group
      socket.listenMulticastGroup("230.0.0.1", asyncResult2 -> {
          if (asyncResult2.succeeded()) {
            // will now receive packets for group

            // do some work

            socket.unlistenMulticastGroup("230.0.0.1", asyncResult3 -> {
              System.out.println("Unlisten succeeded? " + asyncResult3.succeeded());
            });
          } else {
            System.out.println("Listen failed" + asyncResult2.cause());
          }
      });
    } else {
      System.out.println("Listen failed" + asyncResult.cause());
    }
});
```
#### Blocking multicast

Beside unlisten a Multicast address it’s also possible to just block multicast for a specific sender address.

Be aware this only work on some Operating Systems and kernel versions. So please check the Operating System documentation if it’s supported.

This an expert feature.

To block multicast from a specific address you can call blockMulticastGroup(…​) on the DatagramSocket like shown here:
```
DatagramSocket socket = vertx.createDatagramSocket(new DatagramSocketOptions());

// Some code

// This would block packets which are send from 10.0.0.2
socket.blockMulticastGroup("230.0.0.1", "10.0.0.2", asyncResult -> {
  System.out.println("block succeeded? " + asyncResult.succeeded());
});
```
### DatagramSocket properties

When creating a DatagramSocket there are multiple properties you can set to change it’s behaviour with the DatagramSocketOptions object. Those are listed here:

* setSendBufferSize Sets the send buffer size in bytes.

* setReceiveBufferSize Sets the TCP receive buffer size in bytes.

* setReuseAddress If true then addresses in TIME_WAIT state can be reused after they have been closed.

* setTrafficClass

* setBroadcast Sets or clears the SO_BROADCAST socket option. When this option is set, Datagram (UDP) packets may be sent to a local interface’s broadcast address.

* setMulticastNetworkInterface Sets or clears the IP_MULTICAST_LOOP socket option. When this option is set, multicast packets will also be received on the local interface.

* setMulticastTimeToLive Sets the IP_MULTICAST_TTL socket option. TTL stands for "Time to Live," but in this context it specifies the number of IP hops that a packet is allowed to go through, specifically for multicast traffic. Each router or gateway that forwards a packet decrements the TTL. If the TTL is decremented to 0 by a router, it will not be forwarded.

### DatagramSocket Local Address

You can find out the local address of the socket (i.e. the address of this side of the UDP Socket) by calling localAddress. This will only return an InetSocketAddress if you bound the DatagramSocket with listen(…​) before, otherwise it will return null.

### Closing a DatagramSocket

You can close a socket by invoking the close method. This will close the socket and release all resources

