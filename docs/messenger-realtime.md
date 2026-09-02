# Flivex Messenger — Developer Integration Guide

## Overview

Flivex Messenger is a real-time communication engine built into Gloowflix Cloud.
It provides **text messaging**, **voice messages**, and **audio/video calling** (WebRTC)
through a simple REST API + WebSocket signaling.

Developers can integrate Flivex Messenger into their **web apps**, **React Native apps**,
or any platform that supports HTTP + WebSocket.

---

## Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                      YOUR APPLICATION                         │
│                                                               │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │  Chat UI    │  │  Voice Msg   │  │  Audio/Video Call UI │ │
│  │  (REST API) │  │  (REST+S3)   │  │  (WebSocket+WebRTC) │ │
│  └──────┬──────┘  └──────┬───────┘  └──────────┬───────────┘ │
└─────────┼────────────────┼──────────────────────┼─────────────┘
          │                │                      │
          ▼                ▼                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                  GLOOWFLIX CLOUD API                            │
│                                                                 │
│  REST: https://api.noverfly.com/v1/cloud/messenger/...      │
│  WebSocket: wss://api.noverfly.com/ws                       │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Conversations │  │ Voice Proc. │  │ Call Signaling       │  │
│  │ Messages     │  │ (FFmpeg)    │  │ (WebRTC relay)       │  │
│  │ PostgreSQL   │  │ S3 Storage  │  │ STUN/TURN            │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Authentication

All REST endpoints use **API Key** authentication. Both `gfk_` (Secret) and `gfc_` (Cloud) keys are accepted:

```
X-Api-Key: gfk_YOUR_SECRET_KEY
```

Get your API key from the NoverFly dashboard: **Settings → API Keys → Create Key**.

WebSocket `/ws` supports two auth modes:

**Mode 1 — API key (gfk_ secret + userId):**

```json
{ "type": "auth", "payload": { "apiKey": "gfk_YOUR_SECRET_KEY", "userId": "uuid-of-user" } }
```

**Mode 2 — JWT dashboard:**

```json
{ "type": "auth", "payload": { "token": "your-jwt-token" } }
```

> `gfc_` (Cloud) keys and project-user (`site_user` / `app_user`) JWTs are **refused** on `/ws`. Use `gfk_` secret + `userId`, or a dashboard JWT.

---

## 1. Text Messaging (REST API)

### Create a Conversation

```bash
POST /v1/cloud/messenger/conversations
X-API-Key: gfk_YOUR_SECRET_KEY

{
  "userIdA": "uuid-of-user-alice",
  "userIdB": "uuid-of-user-bob"
}
```

**Response:**
```json
{
  "conversation": {
    "id": "conv-uuid",
    "type": "DIRECT",
    "participants": [
      { "userId": "uuid-alice", "user": { "firstName": "Alice", "avatarUrl": "..." } },
      { "userId": "uuid-bob", "user": { "firstName": "Bob", "avatarUrl": "..." } }
    ]
  }
}
```

> If a direct conversation already exists between these two users, it returns the existing one.

### List Conversations

```bash
GET /v1/cloud/messenger/conversations?userId=uuid-alice&limit=20&cursor=last-conv-id
X-API-Key: gfk_YOUR_SECRET_KEY
```

**Response:**
```json
{
  "conversations": [
    {
      "id": "conv-uuid",
      "lastMessage": { "content": "Hey!", "sender": { "firstName": "Bob" } },
      "unreadCount": 3,
      "lastMessageAt": "2026-03-27T10:00:00Z"
    }
  ]
}
```

### Send a Message

```bash
POST /v1/cloud/messenger/conversations/{conversationId}/messages
X-API-Key: gfk_YOUR_SECRET_KEY

{
  "senderId": "uuid-alice",
  "content": "Hello Bob! How are you?",
  "type": "TEXT"
}
```

**Response:**
```json
{
  "message": {
    "id": "msg-uuid",
    "type": "TEXT",
    "content": "Hello Bob! How are you?",
    "sender": { "id": "uuid-alice", "firstName": "Alice" },
    "createdAt": "2026-03-27T10:05:00Z"
  }
}
```

> A **WebSocket event** `messenger:message` is automatically sent to all participants.

### Get Message History

```bash
GET /v1/cloud/messenger/conversations/{conversationId}/messages?userId=uuid-alice&limit=50
X-API-Key: gfk_YOUR_SECRET_KEY
```

### Reply to a Message

```bash
POST /v1/cloud/messenger/conversations/{conversationId}/messages
{
  "senderId": "uuid-bob",
  "content": "I'm great!",
  "replyToId": "msg-uuid-of-original"
}
```

---

## 2. Voice Messages

### Flow

```
1. Record audio in browser → Blob (WebM/Opus)
2. Upload Blob to your S3 or Gloowflix S3
3. POST /voice with S3 key + duration
4. Gloowflix processes: normalize loudness + convert to AAC + generate waveform
5. WebSocket event → waveform + processed audio URL delivered to recipient
```

### Create a Voice Message

```bash
POST /v1/cloud/messenger/conversations/{conversationId}/voice
X-API-Key: gfk_YOUR_SECRET_KEY

{
  "senderId": "uuid-alice",
  "durationMs": 4200,
  "originalKey": "tenants/tenant-uuid/voice/msg-uuid/original.webm",
  "sizeBytes": 128000
}
```

**Response:**
```json
{
  "message": {
    "id": "msg-uuid",
    "type": "VOICE",
    "voiceMessage": {
      "durationMs": 4200,
      "status": "PROCESSING",
      "waveform": null
    }
  }
}
```

### WebSocket Events (Voice)

When processing completes, all participants receive:

```json
{
  "type": "event",
  "event": "messenger:voice_ready",
  "data": {
    "conversationId": "conv-uuid",
    "messageId": "msg-uuid",
    "status": "READY",
    "processedKey": "tenants/.../voice.aac",
    "waveform": [0.1, 0.4, 0.8, 0.6, 0.3, 0.2]
  }
}
```

### Frontend Audio Player (Example — React)

```tsx
function VoicePlayer({ waveform, audioUrl, duration }: Props) {
  const audioRef = useRef<HTMLAudioElement>(null);
  const [playing, setPlaying] = useState(false);

  return (
    <div className="flex items-center gap-2 p-2 bg-gray-100 rounded-xl">
      <button onClick={() => {
        if (playing) audioRef.current?.pause();
        else audioRef.current?.play();
        setPlaying(!playing);
      }}>
        {playing ? '⏸️' : '▶️'}
      </button>

      {/* Waveform bars */}
      <div className="flex items-end gap-px h-8">
        {waveform.map((v, i) => (
          <div key={i} className="w-1 bg-blue-500 rounded" style={{ height: `${v * 100}%` }} />
        ))}
      </div>

      <span className="text-xs text-gray-500">
        {Math.floor(duration / 60)}:{String(duration % 60).padStart(2, '0')}
      </span>

      <audio ref={audioRef} src={audioUrl} onEnded={() => setPlaying(false)} />
    </div>
  );
}
```

---

## 3. Audio Calls (WebRTC)

Audio calls use **WebRTC** for peer-to-peer audio streaming.
The Gloowflix WebSocket server handles **signaling only** — actual audio goes P2P.

### Step 1: Get RTC Config

```bash
GET /v1/cloud/messenger/rtc-config
X-API-Key: gfk_YOUR_SECRET_KEY
```

**Response:**
```json
{
  "iceServers": [
    { "urls": "stun:stun.l.google.com:19302" },
    { "urls": "turn:turn.gloowflix.cloud:3478", "username": "...", "credential": "..." }
  ],
  "signalEndpoint": "wss://api.noverfly.com/ws"
}
```

### Step 2: Connect WebSocket & Authenticate

```javascript
const ws = new WebSocket('wss://api.noverfly.com/ws');

ws.onopen = () => {
  ws.send(JSON.stringify({
    type: 'auth',
    payload: { apiKey: 'gfk_YOUR_SECRET_KEY', userId: 'uuid-of-caller' }
  }));
};
```

### Step 3: Initiate Call

```javascript
// Caller sends:
ws.send(JSON.stringify({
  type: 'call:initiate',
  payload: {
    targetUserId: 'uuid-of-bob',
    callType: 'AUDIO',
    tenantId: 'tenant-uuid'
  }
}));

// Server responds with callId:
// { type: "event", event: "call:created", data: { callId: "call-uuid", ... } }

// Callee receives:
// { type: "event", event: "call:incoming", data: { callId: "call-uuid", callerId: "uuid-alice", callType: "AUDIO" } }
```

### Step 4: WebRTC Negotiation

```javascript
// ─── CALLER SIDE ───────────────────────────────
const pc = new RTCPeerConnection({ iceServers });

// Get microphone
const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
stream.getTracks().forEach(track => pc.addTrack(track, stream));

// Create and send offer
const offer = await pc.createOffer();
await pc.setLocalDescription(offer);

ws.send(JSON.stringify({
  type: 'call:offer',
  payload: {
    targetUserId: 'uuid-bob',
    callId: 'call-uuid',
    sdp: { type: offer.type, sdp: offer.sdp }
  }
}));

// Send ICE candidates as they arrive
pc.onicecandidate = (e) => {
  if (e.candidate) {
    ws.send(JSON.stringify({
      type: 'call:ice',
      payload: {
        targetUserId: 'uuid-bob',
        callId: 'call-uuid',
        candidate: e.candidate.toJSON()
      }
    }));
  }
};

// Play remote audio
pc.ontrack = (e) => {
  const audio = new Audio();
  audio.srcObject = e.streams[0];
  audio.play();
};
```

```javascript
// ─── CALLEE SIDE ───────────────────────────────
ws.onmessage = async (event) => {
  const msg = JSON.parse(event.data);

  if (msg.event === 'call:incoming') {
    // Show ringing UI...
    // User accepts → start WebRTC
  }

  if (msg.event === 'call:offer') {
    const pc = new RTCPeerConnection({ iceServers });
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    stream.getTracks().forEach(track => pc.addTrack(track, stream));

    await pc.setRemoteDescription(msg.data.sdp);
    const answer = await pc.createAnswer();
    await pc.setLocalDescription(answer);

    ws.send(JSON.stringify({
      type: 'call:answer',
      payload: {
        targetUserId: msg.data.callerId,
        callId: msg.data.callId,
        sdp: { type: answer.type, sdp: answer.sdp }
      }
    }));

    pc.onicecandidate = (e) => {
      if (e.candidate) {
        ws.send(JSON.stringify({
          type: 'call:ice',
          payload: {
            targetUserId: msg.data.callerId,
            callId: msg.data.callId,
            candidate: e.candidate.toJSON()
          }
        }));
      }
    };

    pc.ontrack = (e) => {
      const audio = new Audio();
      audio.srcObject = e.streams[0];
      audio.play();
    };
  }

  if (msg.event === 'call:ice') {
    await pc.addIceCandidate(msg.data.candidate);
  }
};
```

### Step 5: Hang Up

```javascript
ws.send(JSON.stringify({
  type: 'call:hangup',
  payload: {
    targetUserId: 'uuid-bob',
    callId: 'call-uuid',
    reason: 'hangup'         // hangup | timeout | error
  }
}));

// Clean up
pc.close();
stream.getTracks().forEach(t => t.stop());
```

---

## 4. Video Calls (WebRTC)

Video calls are **identical** to audio calls — just add `video: true`:

```javascript
// Only difference from audio:
const stream = await navigator.mediaDevices.getUserMedia({
  audio: true,
  video: true    // ← add this
});

// When initiating:
ws.send(JSON.stringify({
  type: 'call:initiate',
  payload: {
    targetUserId: 'uuid-bob',
    callType: 'VIDEO',    // ← VIDEO instead of AUDIO
    tenantId: 'tenant-uuid'
  }
}));

// Display remote video:
pc.ontrack = (e) => {
  const videoElement = document.getElementById('remote-video');
  videoElement.srcObject = e.streams[0];
};
```

### Toggle Camera / Mute During Call

```javascript
// Mute microphone
const audioTrack = stream.getAudioTracks()[0];
audioTrack.enabled = false; // muted
audioTrack.enabled = true;  // unmuted

// Toggle camera
const videoTrack = stream.getVideoTracks()[0];
videoTrack.enabled = false; // camera off
videoTrack.enabled = true;  // camera on

// Switch to screen share
const screenStream = await navigator.mediaDevices.getDisplayMedia({ video: true });
const screenTrack = screenStream.getVideoTracks()[0];
const sender = pc.getSenders().find(s => s.track?.kind === 'video');
await sender.replaceTrack(screenTrack);

// Switch back to camera
screenTrack.onended = async () => {
  const camStream = await navigator.mediaDevices.getUserMedia({ video: true });
  await sender.replaceTrack(camStream.getVideoTracks()[0]);
};
```

---

## 5. React Native Integration

For native mobile apps (React Native), use `react-native-webrtc`:

```bash
npm install react-native-webrtc react-native-callkeep @react-native-firebase/messaging
```

```typescript
import {
  RTCPeerConnection,
  RTCSessionDescription,
  RTCIceCandidate,
  mediaDevices,
  RTCView,
} from 'react-native-webrtc';

// Same WebSocket + signaling flow as web
const ws = new WebSocket('wss://api.noverfly.com/ws');

// Get camera + mic (same API as web)
const stream = await mediaDevices.getUserMedia({ audio: true, video: true });

// Create peer connection (same API as web)
const pc = new RTCPeerConnection({ iceServers });
stream.getTracks().forEach(track => pc.addTrack(track, stream));

// Display video (React Native component instead of <video>)
// <RTCView streamURL={remoteStream.toURL()} style={{ flex: 1 }} />
```

### Native Ringing (CallKeep)

```typescript
import RNCallKeep from 'react-native-callkeep';

// When receiving call:incoming push notification
RNCallKeep.displayIncomingCall(
  callId,           // UUID
  callerName,       // "Alice Dupont"
  callerName,       // display name
  'generic',        // handle type
  callType === 'VIDEO',  // has video
);

// User answers via native phone UI
RNCallKeep.addEventListener('answerCall', ({ callUUID }) => {
  // Connect WebSocket → signaling → WebRTC
});

// User rejects via native phone UI
RNCallKeep.addEventListener('endCall', ({ callUUID }) => {
  ws.send(JSON.stringify({ type: 'call:reject', payload: { targetUserId, callId: callUUID } }));
});
```

---

## 6. WebSocket Events Reference

### Events your app receives:

| Event | Description | Data |
|-------|-------------|------|
| `messenger:message` | New text message | `{ conversationId, message }` |
| `messenger:voice_message` | New voice (processing) | `{ conversationId, message }` |
| `messenger:voice_ready` | Voice processed | `{ conversationId, messageId, waveform, processedKey }` |
| `messenger:read` | Conversation read | `{ conversationId, readBy, readAt }` |
| `messenger:message_deleted` | Message deleted | `{ conversationId, messageId }` |
| `call:incoming` | Incoming call | `{ callId, callerId, callType }` |
| `call:created` | Call log created | `{ callId, calleeId, callType }` |
| `call:offer` | SDP offer | `{ callId, callerId, sdp }` |
| `call:answer` | SDP answer | `{ callId, calleeId, sdp }` |
| `call:ice` | ICE candidate | `{ callId, fromUserId, candidate }` |
| `call:hangup` | Call ended | `{ callId, fromUserId, reason }` |
| `call:reject` | Call rejected | `{ callId, fromUserId }` |
| `call:busy` | Callee busy | `{ callId, fromUserId }` |

### Messages your app sends:

| Type | Payload |
|------|---------|
| `auth` | `{ token: "jwt" }` |
| `call:initiate` | `{ targetUserId, callType, tenantId }` |
| `call:offer` | `{ targetUserId, callId, sdp }` |
| `call:answer` | `{ targetUserId, callId, sdp }` |
| `call:ice` | `{ targetUserId, callId, candidate }` |
| `call:hangup` | `{ targetUserId, callId, reason? }` |
| `call:reject` | `{ targetUserId, callId }` |
| `call:busy` | `{ targetUserId, callId }` |

---

## 7. REST API Reference

### Base URL
```
https://api.noverfly.com/v1/cloud/messenger
```

### Authentication
```
X-API-Key: gfk_YOUR_SECRET_KEY
```

### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/rtc-config` | Get STUN/TURN servers for WebRTC |
| `POST` | `/conversations` | Create direct conversation |
| `GET` | `/conversations?userId=` | List user's conversations |
| `POST` | `/conversations/:id/messages` | Send text message |
| `GET` | `/conversations/:id/messages?userId=` | Get message history |
| `POST` | `/conversations/:id/voice` | Create voice message |
| `GET` | `/calls?userId=` | Get call history |

---

## 8. NoverFly Frontend Integration

NoverFly uses the same API internally. From the NoverFly dashboard (`app.noverfly.com`):

```typescript
// Authenticated user — uses JWT (not API key)
const response = await fetch('/v1/tenants/{tenantId}/messenger/conversations', {
  headers: { Authorization: `Bearer ${jwt}` },
});

// WebSocket — same as developer flow
const ws = new WebSocket('wss://api.noverfly.com/ws');
ws.send(JSON.stringify({ type: 'auth', payload: { token: jwt } }));
```

NoverFly routes (internal, JWT-authenticated):

| Method | Endpoint |
|--------|----------|
| `POST` | `/v1/tenants/:tenantId/messenger/conversations` |
| `GET` | `/v1/tenants/:tenantId/messenger/conversations` |
| `GET` | `/v1/tenants/:tenantId/messenger/conversations/:id` |
| `POST` | `/v1/tenants/:tenantId/messenger/conversations/:id/messages` |
| `GET` | `/v1/tenants/:tenantId/messenger/conversations/:id/messages` |
| `POST` | `/v1/tenants/:tenantId/messenger/conversations/:id/read` |
| `DELETE` | `/v1/tenants/:tenantId/messenger/messages/:id` |
| `POST` | `/v1/tenants/:tenantId/messenger/conversations/:id/voice` |
| `GET` | `/v1/tenants/:tenantId/messenger/calls` |

---

## 9. Bandwidth & Performance

| Feature | Server Load | Client Bandwidth |
|---------|-------------|------------------|
| Text messages | ~1 KB per message (DB + WS) | Negligible |
| Voice message | ~200 KB processed (S3 + FFmpeg 2s) | Upload: ~1 MB, Download: ~200 KB |
| Audio call P2P | **Zero** (signaling only: ~5 KB) | 80-100 kbps per direction |
| Video call 720p P2P | **Zero** (signaling only) | 1.5-2 Mbps per direction |
| Video call via TURN | ~2 Mbps relay | Same as P2P |

---

## 10. Rate Limits

| Endpoint | Limit |
|----------|-------|
| Send message | 60/min per user |
| Create conversation | 10/min per user |
| Voice message | 20/min per user |
| Call initiate | 5/min per user |
| API (developer) | 200/min per API key |
