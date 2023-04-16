# AutonatV2: spec


| Lifecycle Stage | Maturity                 | Status | Latest Revision |
|-----------------|--------------------------|--------|-----------------|
| 1A              | Working Draft            | Active | r2, 2023-04-15  |

Authors: [@sukunrt]

Interest Group: [@marten-seemann], [@marcopolo], [@mxinden]

[@sukunrt]: https://github.com/sukunrt
[@marten-seemann]: https://github.com/marten-seemann
[@mxinden]: https://github.com/mxinden
[@marcopolo]: https://github.com/marcopolo


## Overview

A priori, a node cannot know if it is behind a NAT / firewall or if it is
publicly reachable. Knowing its NAT status is essential for the node to be
well-behaved in the network: A node that's behind a NAT / firewall doesn't need
to advertise its (undialable) addresses to the rest of the network, preventing
superfluous dials from other peers. Furthermore, it might actively seek to
improve its connectivity by finding a relay server, which would allow other
peers to establish a relayed connection.

In `autonat v2` client sends a priority ordered list of addresses. On receiving
this list the server dials the first address on the list that it is capable of
dialing. `autonat v2` allows nodes to determine reachability for individual
addresses. Using `autonat v2` nodes can build an address pipeline where they can
test individual addresses discovered by different sources like identify, upnp
mappings, circuit addresses etc for reachability. Having a priority ordered list
of addresses provides the ability to verify low priority addresses.
Implementations can generate low priority address guesses and add them to
requests for high priority addresses as a nice to have. 

Compared to `autonat v1` there are two major differences
1. `autonat v2` allows testing reachability for an individual address. `autonat
   v1` allowed testing reachability for the node. 
2. `autonat v2` provides a mechanism for nodes to verify whether the peer
actually successfully dialled an address.
3. `autonat v2` provides a mechanism for nodes to dial an ip address different
from the requesting nodes observed ip address without risking amplification
attacks. `autonat v1` disallowed such dials to prevent amplification attacks.


## AutoNAT V2 Protocol

A node wishing to determine reachability of its adddresses sends a `DialRequest`
message to a peer on a stream with protocol ID
`/libp2p/autonat/2.0.0/dial-request`. This `DialRequest` message has a list of
`AddressDialRequest`s. Each item in this list contains an address and a fixed64
nonce. The list is ordered in descending order of priority for verfication.

Upon receiving this message the peer selects the first address from the list
that it is capable of dialing. If this address has an ip address different from
the requesting nodes observed ip address, peer initiates the Amplification
attack prevention mechanism (see [Amplification Attack
Prevention](#amplification-attack-prevention) ). On completion, the peer
proceeds to the next step. If the selected address has the same ip address as
the requesting nodes observed ip address, peer directly proceeds to the next
step skipping Amplification Attack prevention steps.

The peer dials the selected address, opens a stream with Protocol ID
`/libp2p/autonat/2.0.0/dial-attempt` and sends a `DialAttempt` message with the
nonce received in the corresponding `AddressDialRequest`. The peer MUST dial the
first address in the list it is capable of dialing. The peer MUST NOT dial any
address not in the list of addresses sent in the request. The peer MUST ignore
errors on `/libp2p/autonat/2.0.0/dial-attempt` stream

Upon completion of the dial attempt, the peer sends a `DialResponse` message to
the initiator node on the `/libp2p/autonat/2.0.0/dial-request` stream with the
address that it attempted to dial and the appropriate `ResponseStatus`. see
[Requirements For ResponseStatus](#requirements-for-responsestatus)

The initiator SHOULD check that the nonce received in the `DialAttempt` is the
same as the nonce the initiator sent in the `AddressDialRequest` for the address
received in `DialResponse`. If the nonce is different, the initiator MUST
discard this response.


### Requirements for ResponseStatus

On receiving a `DialRequest` the peer selects the first address on the list it
is capable of dialing. The `ResponseStatus` sent by the peer in the
`DialResponse` message MUST be set according to the following requirements

`OK`: the peer was able to dial the selected address successfully.

`E_DIAL_ERROR`: the peer attempted to dial the selected address and was unable
to connect. 

`E_DIAL_REFUSED`: the peer didn't attempt a dial because of rate limiting,
resource limit reached or blacklisting.

`E_TRANSPORT_NOT_SUPPORTED`: the peer didn't have the ability to dial any of the
requested addresses.

`E_BAD_REQUEST`: the peer didn't attempt a dial because it was unable to decode
the message. This includes inability to decode the requested address to dial.

`E_INTERNAL_ERROR`: error not classified within the above error codes occured on
peer that prevented it from completing the request.


### Amplification Attack Prevention

When a client asks a server to dial an address that is not the clients observed
ip address, the server asks the client to send him some non trivial amount of
bytes as a cost to dial a different ip address. The number of bytes is decided
such that it's sufficiently larger than a new connection handshake cost. This
makes amplification attacks unattractive.

On receiving a `DialRequest`, the server selects the first address it is capable
of dialing. If this selected address has a ip different from the clients
observed ip, the server sends a `DialDataRequest` message with `numBytes` set to
a sufficiently large value on the `/libp2p/autonat/2.0.0/dial-request` stream

Upon receiving a `DialDataRequest` message the client responds with
`DialDataResponse` message with a boolean `accepted` indicating if it accepted
the request. If `accepted` is `false`, the `DialRequest` is considered complete
and both the client and the server close the stream. If `accepted` is `true`,
client starts transferring `numBytes` bytes to the server. 

The server on receiving `numBytes` bytes attempts to dial the address. 


## Implementation Suggestions

For any given address, implementations SHOULD do the following
- periodically recheck reachability status
- query multiple peers to determine reachability

The suggested heuristic for implementations is to consider an address reachable
if more than 3 peers report a successful dial and to consider an address
unreachable if more than 3 peers report unsuccessful dials. 

Implementations are free to use different heuristics than this one 


## RPC Messages

All RPC messages sent over a stream are prefixed with the message length in
bytes, encoded as an unsigned variable length integer as defined by the
[multiformats unsigned-varint spec][uvarint-spec]. 

All RPC messages are of type `Message`. A `DialRequest` message is sent as a
`Message` with the `dialRequest` field set and the `type` field set to
`DIAL_REQUEST`. Other message types are handled similarly.


```proto
syntax = "proto3";

message Message {
  enum MessageType {
    DIAL_REQUEST       = 0;
    DIAL_RESPONSE      = 1;
    DIAL_ATTEMPT       = 2;
    DIAL_DATA_REQUEST  = 3;
    DIAL_DATA_RESPONSE = 4;
  }

  MessageType type = 1;
  DialRequest dialRequest = 2;
  DialResponse dialResponse = 3;
  DialAttempt dialAttempt = 4;
  DialDataRequest dialDataRequest = 5;
  DialDataResponse dialDataResponse = 6;
}

message AddressDialRequest {
  bytes addr = 1;
  fixed64 nonce = 2;
}

message DialRequest {
  repeated AddressDialRequest addressDialRequests = 1;
}

message DialAttempt {
  fixed64 nonce = 1;
}

message DialDataRequest {
  uint64 numBytes = 1;
}

message DialDataResponse {
  bool accepted = 1;
}

message DialResponse {
  enum ResponseStatus {
      OK                        = 0;
      E_DIAL_ERROR              = 100;
      E_DIAL_REFUSED            = 101;
      E_TRANSPORT_NOT_SUPPORTED = 102;
      E_BAD_REQUEST             = 200;
      E_INTERNAL_ERROR          = 300;
    }

  ResponseStatus status = 1;
  string statusText = 2;
  bytes addr = 3;
}
```

[uvarint-spec]: https://github.com/multiformats/unsigned-varint

