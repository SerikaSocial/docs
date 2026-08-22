# WebRTC P2P transport (M6)

A P2P instance carries the same pose/voice/chat traffic as a dedicated relay, over a WebRTC
data-channel mesh instead of a UDP relay. It sits behind the same `ISerikaTransport` seam, so
nothing above the transport changes — P2P is a *deployment mode*, not a second netcode stack.

## Pieces

| Piece | Where | Role |
|---|---|---|
| ICE discovery | `server/api` `GET /v1/rtc/ice` | Returns `iceServers` (STUN + TURN). TURN worker > env coturn > STUN-only. |
| coturn | `infra/docker-compose.yml` (`coturn`) | TURN/STUN relay for symmetric NATs. Static long-term creds matching `TURN_USERNAME/PASSWORD`. |
| Signalling | `server/gateway` `/gateway` WS | Relays `rtc:join` / `rtc:peers` / `rtc:peer-join` / `rtc:signal` / `rtc:peer-leave`. Holds no media. |
| Transport | `game/Net/WebRtc/WebRtcTransport.cs` | WebRTC mesh implementing `ISerikaTransport`. |

## Connection flow

```
client → api    GET /v1/rtc/ice                      → iceServers (fresh TURN creds)
client → gateway WS connect ?token=<session jwt>
client → gateway {rtc:join, instanceId}
gateway→ client {rtc:peers, peers:[...]}              ← existing members (newcomer = offer initiator)
        for each existing peer:
          new WebRtcPeerConnection + 2 negotiated channels (id1 pose/unreliable, id2 chat/reliable)
          CreateOffer → session_description_created → {rtc:signal, to, data:{sdp}}
gateway→ peer    {rtc:signal, from, data:{sdp}}       → SetRemoteDescription → auto-answer
        ice_candidate_created → {rtc:signal, data:{ice}} ↔ AddIceCandidate
        → channels open → pose/voice/chat flow peer-to-peer, never through the gateway
```

The **newcomer always initiates** offers (gateway `rtc:peers` lists existing members to the
joiner; existing members get `rtc:peer-join` and answer). This keeps offers one-directional and
avoids glare. Channels are **negotiated** (both sides create ids 1 and 2 identically), so no
`data_channel_received` handshake is needed.

## Wire framing

Data-channel packets reuse the relay envelope minus the peer id (the sender is the connection):
`[MsgType:u8][payload]`, where payload is the shared `serika-proto` pose/voice codec, or UTF-8 for
`Chat`. So the codec guarantee (golden corpus) is identical on P2P and dedicated paths.

## Requirements & status

- **Godot needs the `webrtc-native` GDExtension.** Godot ships the WebRTC *classes* but not the
  implementation; without the extension `WebRtcPeerConnection.Initialize` fails and the transport
  reports it via `Rejected` so callers fall back to the dedicated relay.
- `WebRtcTransport` is complete and compiles. Wiring the join flow to *choose* it (when the
  allocator returns a `p2p` instance with a gateway URL) is the remaining integration; today
  `Main` always uses `UdpTransport`. Construct it as:

  ```csharp
  var ice = await api.GetIceServersAsync();       // /v1/rtc/ice
  var transport = new WebRtcTransport(gatewayWsUrl, api.SessionToken, instanceId, ParseIce(ice));
  transport.Connect(null, null);                  // config came via the ctor
  ```

## Env

See `server/.env.example` (WebRTC / P2P section): `STUN_URLS`, `TURN_URL`, `TURN_USERNAME`,
`TURN_PASSWORD`, `TURN_EXTERNAL_IP`, `TURN_WORKER_URL`, `GATEWAY_WS_URL`. Bring coturn up with
`docker compose -f infra/docker-compose.yml up coturn`.
