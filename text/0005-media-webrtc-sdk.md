- Feature Name: media-webrtc-sdk
- Start Date: 2024-02-02
- RFC PR: [8xff/rfcs#0000](https://github.com/8xff/rfcs/pull/0000)

# 1. Summary

[summary]: #summary

This RFC provides an overview of how the WebRTC client connects to atm0s-media-server and its functionality.

# 2. Motivation

[motivation]: #motivation

To enhance the user experience, we aim to develop a client SDK that is platform-agnostic and highly customizable. This SDK should provide a seamless integration with atm0s-media-server network topology and align with our approach to handle media streams. Our primary focus is to ensure that the SDK is user-friendly and easily adaptable to various use cases.

# 3. User Benefit

User can use WebRTC to connect to the media server and create complex media stream topologies.

# 4. Design Proposal

To ensure simplicity, we propose a SDK protocol that exclusively uses HTTP (not WebSocket) and WebRTC.

- HTTP is used only for sending RPC requests to the cluster, typically during the Connect and Retry phases.
- WebRTC is utilized for sending and receiving media streams, as well as RPC and event communication after establishing a connection.
- Each client will have sender tracks and receiver tracks to handle media streams. Senders are responsible for sending media streams to the server, while receivers are used to receive media streams from the server. Both senders and receivers can be paused, resumed, or switched to another source. There are two types of senders: audio and video, and two types of receivers: audio and video. Each sender and receiver is assigned a unique ID by the client, such as 'audio_1', 'audio_2', 'video_1', 'video_2', and it does not need to be unique with other clients.

We have some terms:

- **_Sender_**: A sender is a track that sends media to the server. It can be an audio or video track.
- **_Receiver_**: A receiver is a track that receives media from the server. It can be an audio or video track.
- **_Source_**: A source that the receiver is pinned to. It can be a audio or video. Source is identify by a pair peer_id and track_id.
- **_Peer_**: A peer is a client that is connected to the server.
- **_Room_**: A room is a group of peers that are connected to the same server.
- **_Conn_**: A connection is a WebRTC connection between the client and the server.
- **_Feature_**: A feature is a set of advance functionalities that the client can use. For example, mix-minus, chat, ...

## 4.1 HTTP Request/Response format

**Request and Response Format**

All requests and responses will be encoded in JSON format. The format is described as follows:

**_Body:_**: JSON

**_Response:_**

```
{
    success: bool,
    error_code: Option<String>,
    error_msg: Option<String>,
    data: Option<JSON>,
}
```

## 4.2 Connect Establishment

Before connecting to the server, the client needs to prepare the following:

- Enable WebRTC connection with data channel.
- Create a list of senders and receivers.
- Generate WebRTC OfferSDP.

Once the client is ready, it can send a connect request to the server.

**_Endpoint_**: `POST GATEWAY/webrtc/connect`

**_Headers_**: `Authorization: Bear {token}`

**_Body_**:

```
{
    version: Option<String>,
    room: String,
    peer: String,
    metadata: Option<String>,
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
                    source: Option<{
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
                metadata: Option<String>,
                state: Option<{
                    active: bool,
                    screen: bool,
                }>,
            }
        ],
    },
    sdp: Option<String>
}
```

**_Response Data:_**

```
{
    sdp: Option<String>,
    conn_id: String,
}
```

The explanation of each request parameter:

- version: is the version of the client SDK.
- room_id: is the room that the client wants to connect to. This is [a-z0-9-] string, maximum is 32 characters.
- peer_id: is the ID of the client. It's used to identify the client on the server side. It's unique in the room. This is [a-z0-9-] string, maximum is 32 characters
- metadata: is the metadata of the client. It can be used to store some information about the client, such as user name .... It's optional and should small than 512 characters.
- event:

  - publish: `full` will publish both peer info and track info. `track` will only publish track info.
  - subscribe: `full` will subscribe to both remote peer info and track info. `track` will only subscribe to remote track info. `manual` will not subscribe to any source, the client must do it manually. This feature is useful for clients who want to use manual mode to subscribe to remote tracks. For example, in a proximity based audio application like [Gather](https://www.gather.town/), the client will set it to `manual` and only subscribe to peers that are near to it.

- bitrate:

  - ingress is the bitrate mode for the ingress stream. In `save` mode, the media server will limit the bitrate based on the network and consumers. In `max` mode, the media server will only limit the bitrate based on the network and media server configuration.

- features: a JSON object containing some features that the client wants to use. For example: mix-minus, spatial room, etc.
- tracks:

  - receivers: a list of receivers that the client wants to create. Each receiver is described with:

    - kind: the kind of receiver, audio or video.
    - id: the ID of the receiver.
    - state: the state of the receiver. It's used to restore the receiver state when the client reconnects to the server. It contains:
      - source: the remote source that the client wants to pin to. If it's none, the receiver will be created but not pinned to any source.
      - limit: the limit of the receiver. If it's none, the receiver will be created with the default limit.

  - senders: a list of senders that the client wants to create. Each sender is described with:
    - kind: the kind of sender, audio or video.
    - id: the ID of the sender.
    - uuid: the UUID of the sender. It's used to identify the sender on the client side.
    - metadata: It can be used to store some information about the client's track, such as label name, device .... It's optional and should small than 512 characters.
    - state: the state of the sender. It's used to restore the sender state when the client reconnects to the server. It contains:
      - screen: a flag to indicate whether the sender is screen sharing.
      - pause: a flag to indicate whether the sender is paused.

- sdp: the OfferSDP that the client created.

The explanation of each response parameter:

- sdp: is the AnswerSDP which server created, it should contain all ice-candidates from server.
- conn_id: global identifier of WebRTC connection. This is used by control api like restart-ice, ice-trickle, kick, etc.

Error list:

| Error code            | Description             |
| --------------------- | ----------------------- |
| INVALID_TOKEN         | The token is invalid.   |
| SDP_ERROR             | The sdp is invalid.     |
| INVALID_REQUEST       | The request is invalid. |
| INTERNAL_SERVER_ERROR | The server is error.    |
| GATEWAY_ERROR         | The gateway is error.   |

After that, the client needs to wait for the connected event from the WebRTC connection and the connected event from the data channel.
If the client doesn't receive any event after a period of time, it will set the restart ice flag and retry connecting to the server with the newest offer SDP.
After several attempts (configurable), the client will stop retrying and report an error to the user as CONNECTION_TIMEOUT.

## 4.3 Restart-ice

**_Endpoint_**: POST `GATEWAY/webrtc/:conn_id/restart-ice`

**_Headers_**: `Authorization: Bear {token}`

**_Body_**: same with connect request but the tracks should sending with current state

**_Response Data_**: same with connect response

By doing this, in case of a network change, the client can retry connecting to the server with the newest offer SDP. If the server is still alive, it will respond with a new answer SDP. However, if the server is dead, the gateway will retry connecting to another server. The session state can be restored using the track state and each feature state.

## 4.4 Ice-tricle

Each time the client's WebRTC connection has a new ice-candidate, it should be sent to the gateway using the following endpoint:

**_Endpoint_**: POST `GATEWAY/webrtc/conns/:conn_id/ice-candidate`

**_Body_**: candidate String

**_Response Data_**: None

## 4.5 Datachannel Request/Response format

The format for encoding all requests and responses sent over the data channel is JSON. The structure of the request and response objects is as follows:

Request/Event:

```
{
    type: "event" | "request",
    seq: Number,
    cmd: String,
    data: Option<JSON>,
}
```

Response:

```
{
    type: "answer",
    seq: Number,
    success: bool,
    error_code: Option<String>,
    error_msg: Option<String>,
    data: Option<JSON>,
}
```

The seq is an incremental value generated on the sender side. It helps us to map between requests and responses and also detect data loss.

The cmd is generate with rule: `identify.action`, for example:

- `peer.update_sdp`.
- `sender.{sender_id}.toggle`.
- `receiver.{receiver_id}.switch`.
- `room.peers.subscribe`.
- `room.peers.unsubscribe`.
- `session.disconnect`.
- `session.features.mix_minus.sources.add`.

## 4.6 In-session requests

At the current state, we only have one WebRTC connection to the server, so there is no need to send any requests over HTTP. All requests will be sent over the WebRTC datachannel.

Typically, the client will need to perform various actions with the media server, such as:

- Creating/Releasing senders
- Creating/Releasing receivers
- Sender actions: pause, resume, switch stream
- Receiver actions: pause, resume, switch remote source, update priority, and layers

All actions that involve changing tracks will be performed locally first, and then the `update_sdp` command will be sent to the server.

### 4.6.1 Update SDP

Each time we make changes to the WebRTC connection or negotiationneeded event fired, we need to send an `update_sdp` request to the server over the data channel. This request is described below:

**_Cmd:_**: `peer.update_sdp`

**_Data:_**

```
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
                metadata: Option<String>,
                state: Option<{
                    active: bool,
                    screen: bool,
                }>,
            }
        ],
    }
}
```

**_Response data_**:

```
{
    sdp: String
}
```

### 4.6.2 Room actions, event

We can subscribe to peers event (joined, left, track added, track removed) and also can unsubscribe from it.

#### 4.6.2.1 Subscribe to other peers event

(Note that this action only works with `event.subscribe` manual mode.)

**_Cmd:_** `room.peers.subscribe`

**_Request Data:_**

```
{
    peer_ids: [String],
}
```

**_Response Data:_** None

#### 4.6.2.2 Unsubscribe to other peers event

(Note that this action only works with `subscribe` manual mode.)

**_Cmd:_**: `room.peers.unsubscribe`

**_Request Data:_**

```
{
    peer_ids: [String],
}

```

#### 4.6.2.3 Peer joined event

**_Cmd:_**: `room.peers.{peer}.joined`

**_Event data:_**:

```
{
    metadata: Option<String>,
}
```

**_Response Data:_**: None

#### 4.6.2.3 Peer left event

**_Cmd:_**: `room.peers.{peer}.left`

**_Event data:_**: None

**_Response Data:_**: None

#### 4.6.2.3 Track added event

**_Cmd:_**: `room.peers.{peer}.tracks.{track}.added`

**_Event data:_**:

```
{
    metadata: Option<String>,
    state: {
        active: bool,
        screen: bool,
        simulcast: bool,
    },
}
```

#### 4.6.2.3 Track updated event

**_Cmd:_**: `room.peers.{peer}.tracks.{track}.updated`

**_Event data:_**:

```
{
    state: {
        active: bool,
    },
}
```

#### 4.6.2.3 Track removed event

**_Cmd:_**: `room.peers.{peer}.tracks.{track}.removed`

**_Event data:_**: None

**_Response Data:_**: None

### 4.6.3 Session actions, event

#### 4.6.3.1 Disconnect

**_Cmd:_**: `session.disconnect`

**_Request data:_**: None

**_Response:_**: None

#### 4.6.3.2 Goaway event

Goaway event is sent by the server in some cases:

- The server is going to shutdown or restart.
- The client lifetime is expired, this is useful in some video conference application where each client only has limited session time; example 1 hour.
- The client is kicked by the server.

In case of a server shutdown or restart, the client should reconnect by sending restart-ice request.

**_Cmd:_**: `session.on_goaway`

**_Event data:_**:

```
{
    reason: "shutdown" | "kick",
    message: Option<String>,
    remain_seconds: Number,
}
```

**_Response Data:_**: None

`remain_seconds` is the time that the client has to reconnect to the server. If it's 0, the client should reconnect immediately.

In case of "shutdown", the client should reconnect by sending restart-ice request.

### 4.6.4 Session Sender create/release, actions, events

For creating a sender, we need to create a transceiver with kind as audio or video. After that, we need to create a track and add it to the transceiver. Then we need to send an `update_sdp` request to the server.

For destroying a sender, we need to remove the track from the transceiver and remove the transceiver from the connection. Then we need to send an `update_sdp` request to the server.

Each sender has some actions and events with the following rule: `session.sender.{id}.{action}`

#### 4.6.4.1 Switch sender source

**_Cmd:_**: `session.senders.{id}.toggle`

**_Request data:_**

```
{
    track: Option<String>,
    metadata: Option<String>,
    state: {
        active: bool,
    }
}
```

**_Response Data:_**: None

If the track is none, the sender will be switched to an inactive state, and other clients will receive a track removed event. In case a client needs to deactivate the sender, it should set 'active' to false; this is useful for the mic mute feature.

#### 4.6.4.2 State event

**_Cmd:_**: `session.senders.{id}.state`

**_Event data:_**:

```
{
    state: "new" | "live" | "paused"
}
```

### 4.6.5 Session Receiver create/release, actions

To create a receiver, we need to create a transceiver with kind as audio or video. After that, we need to create a track and add it to the transceiver. Then we need to send an `update_sdp` request to the server.

Each receiver has some actions and events with the following rule: `session.receiver.{id}.{action}`

### 4.6.5.1 Switch receiver source

**_Cmd:_**: `session.receivers.{id}.switch`

**_Request data:_**

````
{
    priority: u16,
    source: Option<{
        peer: String,
        track: String,
    }>,
}

***Response data:***: None

If source is none, the receiver will be paused.

### 4.6.5.2 Limit receiver bitrate

***Cmd:***: `session.receiver.{id}.limit`

***Request data:***

```
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

**_Event:_**: `session.receivers.{id}.state`

**_Event data:_**:

```
{
    state: "no_source" | "live" | "key_only" | "inactive",
    source: Option<{
        peer: String,
        track: String,
        scaling: "single" | "simulcast" | "svc",
        spatials: Number,
        temporals: Number,
        codec: "opus" | "vp8" | "vp9" | "h264" | "h265" | "av1",
    }>,
}
```

Receiver state is explained below:

- `no_source`: The receiver is created but not pinned to any source.
- `live`: The receiver is live.
- `key_only`: The receiver is live but only receives key frames, which may be for speed limiting purposes.
- `inactive`: The receiver is pinned but does not have enough bandwidth to receive.

### 4.6.5.3 Receiver stats event

**_Event:_**: `session.receivers.{id}.stats`

**_Event data:_**:

```
{
    source: Option<{
        bitrate: Number,
        rtt: Number,
        lost: Number,
        jitter: Number,
    }>,
    transmit: Option<{
        spatial: Number,
        temporal: Number,
        bitrate: Number,
    }>,
}
```

## 4.7 Features

### 4.7.1 Feature: mix-minus mixer

The mix-minus feature has two modes:

- Manual: In this mode, the client can manually add or remove sources to the mixer.
- Auto: In this mode, the media server will automatically add or remove all audio sources except the local source to the mixer.

#### 4.7.1.1 Connect request

In connect request, we add field to features params:

```
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

**_Cmd:_**: `session.features.mix_minus.sources.add`

**_Request data:_**

```
{
    sources: [
        {
            peer: String,
            track: String,
        }
    ]
}
```

**_Response data:_**: None

#### 4.7.1.3 Remove source from mixer

Note that, this action only work with `manual` mode.

**_Cmd:_**: `session.features.mix_minus.sources.remove`

**_Request data:_**

```
{
    sources: [
        {
            peer: String,
            track: String,
        }
    ]
}
```

**_Response data:_**: None

#### 4.7.1.4 Pause mix-minus mixer

**_Cmd:_**: `session.features.mix_minus.pause`

**_Request data:_** None

**_Response data:_**: None

#### 4.7.1.5 Resume mix-minus mixer

**_Cmd:_**: `session.features.mix_minus.resume`

**_Request data:_**: None

**_Response data:_**: None

#### 4.7.1.6 State event

**_Cmd:_**: `session.features.mix_minus.state`

**_Event data:_**

```
{
    slots: [
        {
            source: Option<{
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
