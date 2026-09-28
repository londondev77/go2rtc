# Google Nest

[`new in v1.6.0`](https://github.com/AlexxIT/go2rtc/releases/tag/v1.6.0)

For simplicity, it is recommended to connect the Nest/WebRTC camera to the [Home Assistant](../hass/README.md). 
But if you can somehow get the below parameters, Nest/WebRTC source will work without Home Assistant.

```yaml
streams:
  nest-doorbell: nest:?client_id=***&client_secret=***&refresh_token=***&project_id=***&device_id=***
```

## Changes in this fork

This fork carries a set of fixes for Nest/WebRTC sources that are not (yet) in upstream go2rtc. They are gated on the `nest/webrtc` producer, so other WebRTC sources behave as before.

**Streams no longer die every 5 minutes.** Upstream stored the per-stream session (device, `mediaSessionId`, expiry, extend timer) on the one `*API` object shared by every camera using the same credentials. Each dial overwrote the previous camera's session and the extend timer fired only once, so no stream was ever extended and every Nest producer was re-dialed at Google's 5-minute expiry. Each stream now owns its session and keep-alive, which re-arms after every successful `ExtendStream` and refreshes the OAuth token if an extension gets a 401 (streams outlive the hour-long token). Fixes upstream #2108 and #2319.

**Periodic keyframes.** go2rtc's 2-second keyframe request (RTCP PLI) only applied to passive WebRTC producers such as WHIP. Nest is an active-pull producer, so its keyframe interval drifted long when idle and any consumer joining mid-GOP waited up to a whole interval to start. The request is now enabled for Nest, and a FIR (RFC 5104) is sent alongside the PLI because Google was measured ignoring bare PLI on some cameras (upstream #2365, #2368). PLI and FIR are media-plane RTCP, so this does not touch the SDM API quota. Measured on a Nest doorbell: keyframe interval ~3.3 s → ~1.8 s, on-demand `frame.jpeg` grabs ~1 s.

**SPS/PPS in the RTSP SDP.** Google's SDP answer carries `profile-level-id` but no `sprop-parameter-sets` (WebRTC sends SPS/PPS in-band), and go2rtc copied that line verbatim into its RTSP DESCRIBE. RTSP clients such as ffmpeg therefore could not learn the video dimensions until an in-band keyframe arrived, forcing a large `-probesize` and slow opens. SPS/PPS are now captured from the incoming RTP (including STAP-A bundles) and appended to the codec's fmtp line once, so RTSP, MSE and MP4 consumers all see them.

**Requests are throttled to Google's documented limits.** Every Device Access call counts against the project's quota ([limits](https://developers.google.com/nest/device-access/project/limits): `devices.executeCommand` 10 QPM per project and user, each trait command 5 QPM per device, cameras 30 QPM or 100 QPH, `devices.list` 5 QPM), including the ones that come back 429. Upstream's answer to #1397 was to sleep 30 s and then 60 s after a 409 or 429, which keeps a single stream from spinning but does nothing to avoid the quota and leaves every consumer of every stream waiting out the sleep. This fork books a slot in each applicable sliding window before sending a request. A dial (`GenerateWebRtcStream`, `GenerateRtspStream`) that would have to wait more than 5 s fails at once with a `ThrottledError`, so a fallback source serves the consumer instead; keep-alives (`Extend*Stream`) may wait up to 50 s for a slot. Keep-alives (`Extend*Stream`) leave three of the ten per-minute project slots free for dials, and each stream's first extension is pulled forward by a per-device offset of up to two minutes so that sessions dialed together at start-up do not extend together every cycle. A genuine 429 holds the whole project off for a minute. The verdict on a switched-off camera (400 `FAILED_PRECONDITION`) is remembered per device: for one minute the first time, doubling on every repeat up to 5 minutes, and reset as soon as a dial succeeds. A dashboard polling an off camera every few seconds spends no quota at all, a camera that has been off all day is asked twelve times an hour, and a camera that comes back is picked up within five minutes. The log says how long each hold-off is (`nest: ...: not asking Google again for 8m0s`).

**A camera that is off fails fast.** Google answers `400 FAILED_PRECONDITION "The camera is not available for streaming"` for a camera the Google Home app has switched off (for example by presence sensing). Upstream retried that with 30 s and 60 s sleeps, so every consumer of the stream hung for 90 s before failing. A definitive 4xx now returns a typed `StatusError` immediately; only transport errors and 5xx are retried, and 401/409/429 keep their existing token-refresh and back-off handling. The response body, which is the only place Google states the reason, is included in the error. This pairs well with a fallback source so an off camera shows a placeholder instead of an error:

```yaml
streams:
  garage:
    - nest:?client_id=***&client_secret=***&refresh_token=***&project_id=***&device_id=***
    - ffmpeg:/config/standby.jpg#input=-loop 1 -framerate 2 -i /config/standby.jpg#video=h264#raw=-tune stillimage
```

**A rejected offer is retried, not held against the camera.** Google intermittently answers a dial for a camera that is streaming fine with `400 INVALID_ARGUMENT "offerSdp contains an invalid value."`; the next dial, with a fresh offer, is accepted. Counting that 400 as a verdict on the device held a healthy camera off for a minute and put its consumers on the fallback, which starved audio-only consumers (a placeholder has no audio) until they timed out. The dial now re-offers after 1 s, and logs the rejected SDP so the cause can be pinned down.

**Preloads keep the primary warm, and keep trying.** A preload consumer only ever attaches to the primary source: with a warm placeholder as the second source it would otherwise settle for the placeholder when the camera's dial was merely rate limited at start-up, and hold the stream there for good with the real camera left cold. Ordinary consumers still fall through to the placeholder on any dial failure, so a viewer sees the slate rather than an error while a camera is switched off or briefly throttled, and gets the camera back on the next request. A preload that cannot be brought up at start-up is retried on a back-off (15 s, 30 s, 1, 2, then every 5 min) instead of being abandoned after one attempt.

**Cold dials no longer hang.** `Dial` returns right after the SDP answer, before any track has arrived, so the consumer that triggered the dial binds to the first H264 codec in Google's answer (payload type 96). Google may then send on a different payload type; upstream's `OnTrack` matched receivers by codec pointer and created a second receiver, leaving the first consumer attached to one that never received a packet, so `api/frame.jpeg` on an idle stream hung until its timeout. `OnTrack` now reuses the idle early-bound receiver and points its codec at the one actually in use.

**No more corrupt keyframes on attach.** pion's H264 depacketizer appended every FU-A fragment to its buffer whether or not it had seen the start bit, then synthesized a NAL header when the end bit arrived. A consumer attaching mid-NAL got a NAL with a valid IDR type byte and the tail of a slice as its body; `IsKeyframe` accepted it and `api/frame.jpeg` handed it to ffmpeg, which failed with `exit status 183` ("non-intra slice in an IDR NAL unit"). With 2-second keyframes about half of all snapshots failed this way. `RTPDepay` now drops FU-A fragments received before their start bit. Reported upstream as AlexxIT/go2rtc#2490 and pion/rtp#370.

**Packet loss no longer produces smeared keyframes.** WebRTC delivers RTP over UDP and Google does drop packets. A fragment missing from the middle of a 200 KB IDR left a NAL with a hole in it; ffmpeg decoded what it had and repeated the last good rows across the rest of the picture, and every P-frame until the next keyframe referenced the damage. pion's depacketizer only sees payloads and cannot detect this. `RTPDepay` now tracks sequence numbers: on a gap it discards whatever is being assembled and drops access units until one that begins with a keyframe arrives intact. With the 2-second keyframe requests that costs at most a couple of seconds of picture. This is the H264 counterpart of upstream PR #2479 for H265.

**Readable snapshot errors.** When the `frame.jpeg` transcode fails, the error now includes the last lines of ffmpeg's stderr and the NAL unit types and sizes that were handed to it (for example `7:24 8:4 5:156010`) instead of a bare exit status.

**GOP cache.** Add `#gop=1` to the Nest source (or any source) to keep the last GOP in memory so a consumer attaching mid-stream is served the cached keyframe immediately. See the [streams README](../streams/README.md#gop-cache). This is a rebase of upstream PR #1887 by seydx.
