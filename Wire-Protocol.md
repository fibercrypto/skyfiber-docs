# Messages

* [Introduction](#introduction) **INTR** 
* [Get Peers](#get-peers) **GETP** 
* [Give Peers](#give-peers) **GIVP** 
* [Ping](#ping) **PING** 
* [Pong](#pong) **PONG** 

All messages have a 4-byte length prefix, a 4-byte ascii-printable ID prefix, and the message body.  The 4-byte length prefix is equal to the length of the message body plus 4 for the ID prefix.

All array fields are marked with [] and implicitly include a 4-byte unsigned integer prefix.

## Introduction

```
ID: INTR
Fields:
    Mirror  uint32
    Port    uint16
    Version uint32
```

Introduction messages are exchanged when two clients connect.  If no Introduction message is received in 30 seconds, the client terminates the connection, and that peer is banned for 1 hour.   The first message sent in a connection **must** be INTR.  A peer is banned for 8 hours if they violate that rule.

Mirror is a random number generated by a client when started.  In combination with the IP address of the client, it is used to detect both self connections and duplicate connections.  If a self-connection is detected, the client terminates the connection and that peer is banned for 1 hour.

Port identifies the connection's listen port, which should be exchanged with other peers.

At the moment, any clients with mismatched versions immediately disconnect.

A maximum of 1 outgoing connection and 3 total connections are allowed per base IP.  This allows multiple nodes from the same subnet to connect to another node but prevent them from saturating the latter node's resources.

## Get Peers

```
ID: GETP
Fields:
    None
```

A client sends GETP as a request for more peers.  The receiving client is expected to respond with GIVP.

## Give Peers

```
ID: GIVP
Fields:
    Peers []IPAddr
```

GIVP is sent in response to GETP.  IPAddr is a 6 byte struct containing IP address and Port in their integer representations.

When a client receives this message, it adds or updates peers in its peerlist.

## Ping

```
ID: PING
Fields:
    None
```

PING is sent when a client hasn't sent a message to one of its connections in over 30 minutes.  If a client hasn't sent a message in over 90 minutes, the remote side terminates the connection.  It is up to the idling client to send the ping and prevent disconnection.

# Pong

```
ID: Pong
Fields:
    None
```

PONG is sent in reply to PING.