# Realtime and Media

This page documents the realtime, messaging, voice, call, and media-processing capabilities already present in the deployed NoverFly stack.

---

## Realtime Transport

Canonical realtime endpoint:

- `wss://api.noverfly.com/ws`

Compatible alias:

- `wss://api.gloowflix.cloud/ws`

The WebSocket server supports:

- JWT-authenticated dashboard and site-user sessions
- API-key app auth for developer-integrated apps
- tenant-scoped and user-scoped realtime rooms
- call signaling messages for WebRTC flows

After opening the socket, send an auth message within 10 seconds.

### API-key auth example

```json
{
  "type": "auth",
  "payload": {
    "apiKey": "gfk_YOUR_SECRET_KEY",
    "userId": "USER_ID"
  }
}
```

---

## Developer Messenger API

The deployed platform already exposes a developer messenger surface for existing applications.

Auth:

- `X-Api-Key: gfk_...`
- or `X-Api-Key: gfc_...`

Main routes:

| Method | Route | Description |
|---|---|---|
| `GET` | `/v1/cloud/messenger/rtc-config` | Return ICE server config and signal endpoint |
| `POST` | `/v1/cloud/messenger/conversations` | Create a direct conversation |
| `GET` | `/v1/cloud/messenger/conversations` | List a user's conversations |
| `POST` | `/v1/cloud/messenger/conversations/:conversationId/messages` | Send a message |
| `GET` | `/v1/cloud/messenger/conversations/:conversationId/messages` | Read message history |
| `POST` | `/v1/cloud/messenger/conversations/:conversationId/voice` | Create a voice-message record |
| `GET` | `/v1/cloud/messenger/calls` | Read call history |

Typical app flow:

1. Get RTC config from `/v1/cloud/messenger/rtc-config`.
2. Open `/ws` and authenticate.
3. Create or list conversations.
4. Send text messages through the REST API.
5. Upload audio to Cloud storage, then create a voice message record.
6. Use WebSocket call signaling for audio or video calls.

---

## Voice Messages

Voice messages are already part of the deployed messenger stack.

REST route:

- `POST /v1/cloud/messenger/conversations/:conversationId/voice`

Expected body fields include:

- `senderId`
- `durationMs`
- `originalKey`
- `sizeBytes`

There is also an internal voice-processing webhook used by the Flivex service:

- `POST /v1/cloud/messenger/webhooks/voice-processed`

When voice media processing completes, the platform can push realtime updates such as processed waveform data and ready-state notifications.

---

## Audio and Video Calls

WebRTC call signaling is already wired in the WebSocket server.

Current signaling message types include:

- `call:initiate`
- `call:offer`
- `call:answer`
- `call:ice`
- `call:hangup`
- `call:reject`
- `call:busy`

Call logging is already present and the deployed stack tracks:

- call count usage
- total call minutes quotas
- video-call feature availability by plan

Important feature flags and quotas include:

- `messenger_enabled`
- `messenger_video_calls`
- `max_messages_month`
- `max_voice_messages_month`
- `max_calls_month`
- `max_call_minutes_month`

---

## Flivex Media Processing

The Cloud API can create assets for images, video, and audio. When Flivex integration is enabled, the API notifies the Flivex media engine after asset creation.

That media engine can then call back into:

- `POST /webhooks/flivex`

Current Flivex callback event types include:

- `media.probed`
- `media.ready`
- `media.failed`

When media is ready, asset metadata can be enriched with:

- `poster_url`
- `preview_url`
- `playback_url`
- `hls_url`
- `thumbnail_url`
- `renditions`
- `image_variants`

This means the deployed platform already supports processed-media flows for:

- uploaded video
- uploaded audio
- uploaded image assets

---

## Hosted Video and Streaming

The platform already includes routes for hosted video workflows.

Dashboard-managed routes:

| Method | Route | Description |
|---|---|---|
| `POST` | `/v1/tenants/:tenantId/video/connect` | Connect a video provider |
| `POST` | `/v1/tenants/:tenantId/video/test` | Test the video provider connection |
| `POST` | `/v1/sites/:siteId/videos` | Add a video |
| `GET` | `/v1/sites/:siteId/videos` | List videos |
| `POST` | `/v1/sites/:siteId/videos/mp4-url/validate` | Validate a direct MP4 URL |
| `POST` | `/v1/sites/:siteId/videos/mp4-url` | Add a direct MP4 video |
| `POST` | `/v1/public/sites/:siteId/videos/:videoId/access` | Check access to a monetized public video |

The deployed limits model also includes a `video_hosting` feature flag.

---

## Music Streaming

The music module already exposes a track streaming route:

- `GET /v1/tenants/:tenantId/music/tracks/:trackId/stream`

This route returns a stream URL for the requested track.

---

## Cloud Upload and Media Search

The Cloud API already covers the media input side used by apps before messaging or streaming features consume those assets.

Routes already present today:

- `POST /v1/api/cloud/upload`
- `POST /v1/api/cloud/upload/commit`
- `POST /v1/api/cloud/upload/direct`
- `GET /v1/api/cloud/assets`
- `DELETE /v1/api/cloud/assets/:assetId`
- `GET /v1/api/cloud/search/images`
- `GET /v1/api/cloud/search/videos`
- `GET /v1/api/cloud/search/gifs`
- `GET /v1/api/cloud/search/icons`
- `GET /v1/api/cloud/search/3d`
- `POST /v1/api/cloud/import/image`

This lets applications combine:

- their own uploads
- external media discovery
- processed playback URLs
- realtime chat and call flows

---

## Summary

If you need chat, voice, calls, or streaming on NoverFly today:

- use `https://api.noverfly.com`
- use `/v1/cloud/messenger/*` for developer-integrated chat flows
- use `wss://api.noverfly.com/ws` for realtime signaling
- use `/v1/api/cloud/*` for uploads and media input
- rely on Flivex callbacks for processed media outputs
- use the existing video and music route families where dashboard-managed media is needed
