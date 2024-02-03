- Feature Name: media-webrtc-sdk
- Start Date: 2024-02-02
- RFC PR: [8xff/rfcs#0000](https://github.com/8xff/rfcs/pull/0000)

# 1. Summary

[summary]: #summary

This RFC describes how WebRTC client is connect to atm0s-media-server and how it works.

# 2. Motivation

[motivation]: #motivation

We need to create a client sdk which can be used in any platform and can be customized to fit with any use case. This sdk should be easy to use and easy to customize. It also need to work-well with atm0s-media-server network topology and our aproach to handle media stream.

# 3. User Benefit

User can use WebRTC to connect to media-server, and can create very complex media stream topology.

# 4. Design Proposal

For simplicty we proposal a sdk protocol which only use with HTTP (not websocket) and WebRTC.

- HTTP is only used for sending RPC request to cluster (typicaly is Connect and Retry phase).
- WebRTC is used for sending and receiving media stream and also rpc, event after connected.

## 4.1 HTTP Request/Response format

All request and response will be encoded in JSON format. The format is described as below:

**Body:**: JSON

**Response:\***

```json
{
    success: bool,
    error_code: Option<String>,
    error_msg: Option<String>,
    data: Option<JSON>,
}
```

## 4.2 Connect request

Client can prepare some senders or receivers before connect to server. When client connect to server, it will send a connect request to server.

Client must to prepare:

- WebRTC connection with datachannel enabled.
- List of senders and receivers.
- WebRTC OfferSDP.

**_Endpoint_**: `POST GATEWAY/webrtc/connect`

**_Headers_**: `Authorization: Bear {token}`

**_Body_**:

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

**_Response Data:_**

```json
{
    sdp: Option<String>,
    conn_id: String,
}
```

In request:

- version: is the version of client sdk.
- event:

  - publish: `full` will publish both peer info and tracks info. `track` will only publish tracks info.
  - subscribe: `full` will subscribe both remote peer info and tracks info. `track` will only subscribe remote tracks info. `manual` with not subscribe any source, client must do it manual. This feature is useful for client which want to use manual mode to subscribe remote tracks. Example in spatial room application, client will set to `manual` and only subscribe to peer which near to it.

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

In response:

- sdp: is the AnswerSDP which server created, it should contain all ice-candidates from server.
- conn_id: global identify of WebRTC connection. This is used by control api like restart-ice, ice-trickle, kick, etc.

Error list:

| Error code            | Description             |
| --------------------- | ----------------------- |
| INVALID_TOKEN         | The token is invalid.   |
| SDP_ERROR             | The sdp is invalid.     |
| INVALID_REQUEST       | The request is invalid. |
| INTERNAL_SERVER_ERROR | The server is error.    |
| GATEWAY_ERROR         | The gateway is error.   |

After that client need to wait for connected event from WebRTC connection and connected event from datachannel.
If after a period of time, client don't receive any event, it will set restart ice flag and retry connect to server with newest offer-sdp.
After some tries (configurable), client will stop retry and report error to user as CONNECTION_TIMEOUT.

## 4.3 Restart-ice

**_Endpoint_**: POST `GATEWAY/webrtc/:conn_id/restart-ice`

**_Headers_**: `Authorization: Bear {token}`

**_Body_**: same with connect request but the tracks should sending with current state

**_Response Data_**: same with connect response

By that way, incase of network change, client can retry connect to server with newest offer-sdp and if the server is still alive, it will response with new answer-sdp. If the server is dead, client will retry connect to another server and can be restore the session state by using track state.

## 4.4 Ice-tricle

Each time client WebRTC connection has a new ice-candidate, it should sending to gateway over:

**_Endpoint_**: POST `GATEWAY/webrtc/conns/:conn_id/ice-remote`

**_Body_**: candidate String

**_Response Data_**: None

## 4.5 Datachannel Request/Response format

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
- `room.peers.subscribe`.
- `room.peers.unsubscribe`.
- `session.disconnect`.
- `session.features.mix-minus.add_source`.

## 4.6 In-session requests

At current state, we will have only one WebRTC connection to server. So we don't need to send any request to server. All request will be send over datachannel.

Typicaly, client will need some actions with media server:

- Create/Release sender
- Create/Release receiver
- Sender action: pause, resume, switch stream
- Receiver action: pause, resume, switch remote source, update priority and layers

All action which changed streams will be do at local-first, then calling updateSdp to server.

### 4.6.1 UpdateSDP

Each time we changed something in WebRTC connection, we need to send updateSdp request to server over datachannel. The request will be described in below:

**_Cmd:_**: `peer.updateSdp`

**_Data:_**

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

**_Response data_**:

```json
{
    sdp: String
}
```

### 4.6.2 Room actions, event

We can subscribe to peers event (joined, leaved, track added, track removed) and also can unsubscribe from it.

#### 4.6.2.1 Subscribe to other peers event

**_Cmd:_** `room.peers.subscribe`

**_Request Data:_**

```json
{
    peer_id: String,
}
```

**_Response Data:_** None

#### 4.6.2.2 Unsubscribe to other peers event

**_Cmd:_**: `room.peers.unsubscribe`

**_Request Data:_**

```json
{
    peer_id: String,
}

```

**_Response Data:_**: None

### 4.6.3 Session actions, event

#### 4.6.3.1 Disconnect

**_Cmd:_**: `session.disconnect`

**_Request data:_**: None

**_Response:_**: None

### 4.6.4 Session Sender create/release, actions, events

For create a sender we need to create a transiver with kind is audio or video. After that we need to create a track and add it to transiver. Then we need to sending updateSdp request to server.

For destroy a sender, we need to remove track from transiver and remove transiver from connection. Then we need to sending updateSdp request to server.

Each sender has some actions and some event with rule: `session.sender.{id}.{action}`

#### 4.6.4.1 Switch sender source

**_Cmd:_**: `session.sender.{id}.toggle`

**_Request data:_**

```json
{
    track: Option<String>,
    label: Option<String>,
}
```

**_Response Data:_**: None

**_Cmd:_**: `sender_event.{id}.state`

#### 4.6.4.2 State event

**_Event data:_**:

```json
{
    state: "new" | "live" | "paused"
}
```

### 4.6.5 Session Receiver create/release, actions

For create a receiver we need to create a transiver with kind is audio or video. After that we need to create a track and add it to transiver. Then we need to sending updateSdp request to server.

Each receiver has some actions and some event with rule: `session.receiver.{id}.{action}`

### 4.6.5.1 Switch receiver source

**_Cmd:_**: `session.receiver.{id}.switch`

**_Request data:_**

````json
{
    priority: u16,
    remote: Option<{
        peer: String,
        track: String,
    }>,
}

***Response data:***: None

If remote is none, the receiver will be paused.

### 4.6.5.2 Limit receiver bitrate

***Cmd:***: `session.receiver.{id}.limit`

***Request data:***

```json
{
    priority: u16,
    min_spatial: Option<u8>,
    max_spatial: u8,
    min_temporal: Option<u8>,
    max_temporal: u8,
}
````

**_Response data:_**: None

### 4.6.5.3 Receiver state event

**_Event:_**: `session.receiver.{id}.state`

**_Event data:_**:

```json
{
    state: "no_source" | "live" | "key_only" | "inactive",
    source: Option<{
        peer: String,
        track: String,
    }>,
}
```

Receiver state is explain below:

- no_source: The receiver is created but don't pin to any source.
- live: The receiver is live.
- key_only: The receiver is live but only receive key frame, this maybe for speed limiter.
- inactive: The receiver is pinned but not enough bandwidth to receive.

### 4.6.5.3 Receiver stats event

**_Event:_**: `session.receiver.{id}.stats`

**_Event data:_**:

```json
{
    codec: String,
    ingress: Option<{
        scaling: "single" | "simulcast" | "svc",
        spatials: Number,
        temporals: Number,
        bitrate: Number,
        rtt: Number,
        lost: Number,
        jitter: Number,
    }>,
    egress: Option<{
        spatial: Number,
        temporal: Number,
        bitrate: Number,
    }>,
}
```

## 4.7 Features

### 4.7.1 Feature: mix-minus mixer

Mix-minus feature has 2 modes:

- Manual: client can add or remove source to mixer.
- Auto: media-server will automatically add or remove all audio sources except the local source to mixer.

#### 4.7.1.1 Connect request

In connect request, we add field to features params:

```json
{
    features: {
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
}
```

#### 4.7.1.2 Add source to mixer

Note that, this action only work with `manual` mode.

**_Cmd:_**: `session.features.mix-minus.add_source`

**_Request data:_**

```json
{
    peer: String,
    track: String,
}
```

**_Response data:_**: None

#### 4.7.1.3 Remove source from mixer

Note that, this action only work with `manual` mode.

**_Cmd:_**: `session.features.mix-minus.remove_source`

**_Request data:_**

```json
{
    peer: String,
    track: String,
}
```

**_Response data:_**: None

#### 4.7.1.4 Pause mix-minus mixer

**_Cmd:_**: `session.features.mix-minus.pause`

**_Request data:_** None

**_Response data:_**: None

#### 4.7.1.5 Resume mix-minus mixer

**_Cmd:_**: `session.features.mix-minus.resume`

**_Request data:_**: None

**_Response data:_**: None

#### 4.7.1.6 State event

**_Cmd:_**: `session.features.mix-minus.state`

**_Event data:_**

```json
{
    layers: [
        {
            id: string,
            remote: Option<{
                peer: String,
                track: String,
                audio_level: Number,
            }>,
        }
    ]
}
```

# 5. Drawbacks

[drawbacks]: #drawbacks

No drawbacks.

# 6. Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

We have some alternatives:

- Whip/Whep: but it not flexible and cannot be used to create complex media stream topology.
- Livekit protocol: the protocol don't have document and it's is designed for Livekit server topology.

# 7. Unresolved questions

[unresolved-questions]: #unresolved-questions

Not yet.

# 8. Future possibilities

[future-possibilities]: #future-possibilities

We can have some improvements:

- Dynamic event subscription.
- Binary event and request/response format.
