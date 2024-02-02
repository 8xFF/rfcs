- Feature Name: media-webrtc-sdk
- Start Date: 2024-02-02
- RFC PR: [8xff/rfcs#0000](https://github.com/8xff/rfcs/pull/0000)

# Summary
[summary]: #summary

This RFC describes how WebRTC client is connect to atm0s-media-server and how it works.

# Motivation
[motivation]: #motivation

We need to create a client sdk which can be used in any platform and can be customized to fit with any use case. This sdk should be easy to use and easy to customize. It also need to work-well with atm0s-media-server network topology and our aproach to handle media stream.

# User Benefit

User can use WebRTC to connect to media-server, and can create very complex media stream topology.

# Design Proposal

For simplicty we proposal a sdk protocol which only use with HTTP (not websocket) and WebRTC.

- HTTP is only used for sending RPC request to cluster (typicaly is Connect and Retry phase).
- WebRTC is used for sending and receiving media stream and also rpc, event after connected.

### HTTP Request/Response format

All request and response will be encoded in JSON format. The format is described as below:

Request: JSON

Response:

```json
{
    success: bool,
    error_code: Option<String>,
    error_msg: Option<String>,
    data: Option<JSON>,
}
```

### Connect request

Client can prepare some senders or receivers before connect to server. When client connect to server, it will send a connect request to server.

Client must to prepare:

- WebRTC connection with datachannel enabled.
- List of senders and receivers.
- WebRTC OfferSDP.

Endpoint: POST `GATEWAY/webrtc/connect`

Header:
    Authorization: Bear {token}
Body:
```json
{
    version: Option<String>,
    room: String,
    peer: String,
    event: {
        publish: "full" | "track",
        subscribe: "full" | "track" | "manual",
    },
    bitrate: {
        ingress: "save" | "max",
    },
    features: JSON,
    tracks: {
        receivers: [
            {
                kind: "audio" | "video",
                id: String,
                state: Option<{
                    remote: Option<{
                        peer: String,
                        track: String,
                    }>,
                    limit: Option<{
                        priority: u16,
                        min_spatial: Option<u8>,
                        max_spatial: u8,
                        min_temporal: Option<u8>,
                        max_temporal: u8,
                    }>,
                }>
            }
        ],
        senders: [
            {
                kind: "audio" | "video",
                id: String,
                uuid: String,
                label: String,
                state: Option<{
                    screen: bool,
                    pause: bool,
                }>,
            }
        ],
    },
    sdp: Option<String>
}
```

In there:

- version: is the version of client sdk.
- event:
    - publish: full will publish both peer info and tracks info. track will only publish tracks info.
    - subscribe: full will subscribe both remote peer info and tracks info. track will only subscribe remote tracks info. This feature is useful for client which want to use manual mode to subscribe remote tracks. Example in spatial room application, client will set to `manual` and only subscribe to peer which near to it.

- bitrate:
    - ingress is the bitrate mode for ingress stream. In `save` mode, media-server will limit bitrate based on network and consumers. In `max` mode, media-server will only limit bitrate by network and media-server config.

- features: json object for containing some features which client want to use. For example: mix-minus, spatial room, etc.
- tracks:
    - receivers: list of receivers which client want to create. Each receiver is described with:
        - kind: is the kind of receiver, audio or video.
        - id: is the id of receiver.
        - remote: is the remote source which client want to pin to. If it's none, the receiver will be created but not pin to any source.
        - limit: is the limit of receiver. If it's none, the receiver will be created with default limit.

    - senders: list of senders which client want to create. Each sender is described with:
        - kind: is the kind of sender, audio or video.
        - id: is the id of sender.
        - uuid: is the uuid of sender. It's used to identify the sender in client side.
        - label: is the label of sender. It's used to identify the sender in client side.
        - screen: is the flag to indicate that the sender is screen sharing.
- sdp: is the OfferSDP which client created.

After that server will success response with data:

```json
{
    sdp: Option<String>,
    conn_id: String,
}
```

In there:

- sdp: is the AnswerSDP which server created, it should contain all ice-candidates from server.
- conn_id: global identify of WebRTC connection. This is used by control api like restart-ice, ice-trickle, kick, etc.

Error list:

| Error code | Description |
|------------|-------------|
| INVALID_TOKEN | The token is invalid. |
| SDP_ERROR | The sdp is invalid. |
| INVALID_REQUEST | The request is invalid. |
| INTERNAL_SERVER_ERROR | The server is error. |
| GATEWAY_ERROR | The gateway is error. |

After that client need to wait for connected event from WebRTC connection and connected event from datachannel.
If after a period of time, client don't receive any event, it will set restart ice flag and retry connect to server with newest offer-sdp.
After some tries (configurable), client will stop retry and report error to user as CONNECTION_TIMEOUT.

### Restart-ice

Endpoint: POST `GATEWAY/webrtc/:conn_id/restart-ice`
```json
BODY is same with connect request but the tracks should sending with current state
```

By that way, incase of network change, client can retry connect to server with newest offer-sdp and if the server is still alive, it will response with new answer-sdp. If the server is dead, client will retry connect to another server and can be restore the session state by using track state.

### Ice-tricle

Each time client WebRTC connection has a new ice-candidate, it should sending to gateway over:

Endpoint: POST `GATEWAY/webrtc/conns/:conn_id/ice-remote`
Request data: candidate String
Response data: None

### Datachannel Request/Response format

All request and response sending over datachannel will be encoded in JSON format. The format is described as below:

Request/Event:
```json
{
    type: "event" | "request",
    seq: Number,
    cmd: String,
    data: Option<JSON>,
}
```

Response:

```json
{
    type: "answer",
    seq: Number,
    success: bool,
    error_code: Option<String>,
    error_msg: Option<String>,
    data: Option<JSON>,
}
```

The seq is incresemental value, which is generated in sender side. The seq is helped us to mapping between request and response and also detect data lost.

The cmd is generate with rule: `identify.action`, for example:
- `peer.updateSdp`.
- `sender.{sender_id}.toggle`.
- `receiver.{receiver_id}.switch`.

### In-session requests

At current state, we will have only one WebRTC connection to server. So we don't need to send any request to server. All request will be send over datachannel.

Typicaly, client will need some actions with media server:

- Create/Release sender
- Create/Release receiver
- Sender action: pause, resume, switch stream
- Receiver action: pause, resume, switch remote source, update priority and layers

All action which changed streams will be do at local-first, then calling updateSdp to server.

#### UpdateSDP

Each time we changed something in WebRTC connection, we need to send updateSdp request to server over datachannel. The request will be described in below:

request: `peer.updateSdp`
Request data: 
```json
{
    sdp: String,
    tracks: {
        receivers: [
            {
                kind: "audio" | "video",
                id: String,
            }
        ],
        senders: [
            {
                kind: "audio" | "video",
                id: String,
                uuid: String,
                label: String,
                screen: Option<bool>,
            }
        ],
    }
}
```

Response data:
```json
{
    sdp: String
}
```

#### Room actions, event

We can subscribe to peers event (joined, leaved, track added, track removed) and also can unsubscribe from it.

```json
Request: room.peers.subscribe
Request data:
{
    peer_id: String,
}

Response: None
```

```json
Request: room.peers.unsubscribe
Request data:
{
    peer_id: String,
}

Response: None
```

Room have some events:

```json
```

#### Session actions, event

```json
Request: session.disconnect
Request data: None
Response: None
```

#### Session Sender create/release, actions, events

For create a sender we need to create a transiver with kind is audio or video. After that we need to create a track and add it to transiver. Then we need to sending updateSdp request to server. 

For destroy a sender, we need to remove track from transiver and remove transiver from connection. Then we need to sending updateSdp request to server.

Each sender support bellow actions:

```json
Request: session.sender.{id}.toggle
Request data:
{
    track: Option<String>,
    label: Option<String>,
}
Response: None
```

Each sender also fire some event with cmd prefix: "sender_event" json template:

```json
Event: session.sender.{id}.state
Event data: {
    state: "new" | "live" | "paused"
}
```

#### Session Receiver create/release, actions

For create a receiver we need to create a transiver with kind is audio or video. After that we need to create a track and add it to transiver. Then we need to sending updateSdp request to server.

Each receiver support bellow actions:

```json
Request: session.receiver.{id}.switch
Request data:
{
    priority: u16,
    remote: Option<{
        peer: String,
        track: String,
    }>,
}
Response: None
```

If remote is none, the receiver will be paused.


```json
Request: session.receiver.{id}.limit
Request data:
{
    priority: u16,
    min_spatial: Option<u8>,
    max_spatial: u8,
    min_temporal: Option<u8>,
    max_temporal: u8,
}
Response: None
```

Each sender also fire some event with cmd prefix: "sender_event" json template:

```
Event: session.receiver.{id}.state
Event data: "no_source" | "live" | "key_only" | "inactive"
```

```json
Event: session.receiver.{id}.stats
Event data:
{
    codec: String,
    ingress: {
        scaling: "single" | "simulcast" | "svc",
        spatials: Number,
        temporals: Number,
        bitrate: Number,
        rtt: Number,
        lost: Number,
        jitter: Number,
    },
    egress: {
        spatial: Number,
        temporal: Number,
        bitrate: Number,
    },
}
```

Receiver state is explain below:

- no_source: The receiver is created but don't pin to any source.
- live: The receiver is live.
- key_only: The receiver is live but only receive key frame, this maybe for speed limiter.
- inactive: The receiver is pinned but not enough bandwidth to receive.

#### Feature: mix-minus

In connect request, we add field to features params:

```json
{
    mix_minus: {
        mode: "manual" | "auto",
        sources: [
            {
                peer: String,
                track: String,
            }
        ]
    }
}
```

Actions:

```json
Request: session.features.mix-minus.add_source
Request data:
{
    peer: String,
    track: String,
}
Response body: empty
```

```json
Request: session.features.mix-minus.remove_source
Request data:
{
    peer: String,
    track: String,
}
Response body: empty
```

```json
Request: session.features.mix-minus.pause
Request data:
{
    peer: String,
    track: String,
}
Response body: empty
```

```json
Request: session.features.mix-minus.resume
Request data:
{
    peer: String,
    track: String,
}
Response body: empty
```

```json
Event: session.features.mix-minus.state
Event data:
{
    layers: [
        {
            id: string,
            remote: Option<{
                peer: String,
                track: String,
            }>,
            audio_level: Number,
        }
    ]
}
```

# Drawbacks
[drawbacks]: #drawbacks

No drawbacks.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We have some alternatives:

- Whip/Whep: but it not flexible and cannot be used to create complex media stream topology.
- Livekit protocol: the protocol don't have document and it's is designed for Livekit server topology.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Not yet.

# Future possibilities
[future-possibilities]: #future-possibilities

We can have some improvements:

- Dynamic event subscription.
- Binary event and request/response format.