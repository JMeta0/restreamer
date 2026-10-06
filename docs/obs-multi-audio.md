# Sending multiple audio tracks from OBS to Restreamer

Restreamer accepts sources with multiple audio tracks — for example OBS's
"Twitch VOD track" setup, where one audio track goes to the live stream and a
second, different audio track goes to the Twitch VOD (archive). Each
publication target (Twitch, YouTube, Kick, a custom RTMP/SRT endpoint, …) can
select its own ordered set of audio tracks.

This guide explains how to configure OBS so that the audio tracks arrive
**separate** instead of being merged into one stereo mix.

## What "merged audio" means and how to avoid it

OBS mixes audio per **track**, not per source:

- Every source that has the same track checkbox enabled in *Advanced Audio
  Properties* is **summed** into that track. Two sources on track 1 are always
  merged.
- **Simple** output mode streams exactly one audio track: the sum of
  everything enabled on track 1. Any separation you set up is lost.
- **Advanced** output mode can stream several tracks at once. Track separation
  in *Advanced Audio Properties* is the only mechanism that keeps audio apart.

So the recipe is always the same: *Advanced* output mode, one OBS track per
desired output audio track, and no source enabled on two tracks that must stay
separate.

Example: game audio + microphone on the live track, only game audio on the VOD
track:

| Source        | Track 1 (live) | Track 2 (VOD) |
| ------------- | -------------- | ------------- |
| Game capture  | on             | on            |
| Microphone    | on             | off           |

Sources on the same track are summed — that is intended for track 1 here.

## Method A: RTMP with a VOD track (Twitch-style)

This sends **both audio tracks in one RTMP/RTMP(S) stream** using *Enhanced
RTMP* multi-track audio — the same wire format OBS uses for the native Twitch
VOD track.

Requirements:

- OBS Studio **30.2 or newer** (older versions use a Twitch-only signaling
  that Restreamer does not accept)
- Restreamer with FFmpeg 8 (all official bundles since this feature)

Steps:

1. **Settings → Output → Output Mode: Advanced.**
2. **Settings → Advanced Audio Properties**: assign the sources to the OBS
   tracks as described above (track 1 = live audio, track 2 = VOD audio).
3. **Settings → Stream**:
   - Service: **Twitch** — yes, even when Restreamer is the destination. OBS
     only shows the VOD track option for the Twitch service (a data-driven
     option for other services is pending upstream in OBS).
   - Server: **Custom…** → `rtmp://<restreamer-host>:1935/live` (or
     `rtmps://<restreamer-host>:443` if TLS is enabled).
   - Stream key: the **publish token** of the Restreamer channel (everything
     after `rtmp://…/live/` in the publish URL shown by Restreamer).
4. **Settings → Output → Streaming**:
   - Encoder: x264 / a hardware H.264 encoder.
   - Check **Enable Custom Encoder Settings**, then check **Twitch VOD Track**
     and set the VOD track to **track 2**.
   - Select the audio tracks to stream: **1 and 2**.
   - Audio encoder: **AAC** for every selected track (required — the multi
     track mode of OBS only carries AAC), 48 kHz, stereo, 160–320 kbit/s per
     track.

Both tracks arrive as separate audio streams in Restreamer, in the order they
were selected (track 1 first).

## Method B: SRT with MPEG-TS (any number of tracks)

MPEG-TS carries any number of audio tracks natively — no Enhanced RTMP needed.
OBS can send up to six tracks through its *Custom FFmpeg Output*:

1. **Settings → Output → Output Mode: Advanced.**
2. **Settings → Advanced Audio Properties**: assign the sources per OBS track.
3. **Settings → Stream**: Service **Custom…**, Server
   `srt://<restreamer-host>:6000` (the Restreamer SRT port, default 6000).
4. **Settings → Output → Output → Custom FFmpeg Output**:
   - Output: `srt://<restreamer-host>:6000?mode=caller&transtype=live&streamid=<channelid>,mode:publish,token:<publish-token>`
   - Container format: **mpegts**
   - Check every audio track that should be sent. Each checked track becomes
     its own audio stream (PID) — nothing is merged. The bitrate setting is
     applied per track encoder.

Enable the SRT source in the Restreamer channel settings and use the same
`<channelid>` and token as in the SRT stream ID.

## Routing the tracks to your targets

In Restreamer, every publication target has an **"Audio tracks"** list in the
*Source & Encoding* tab:

- Each entry selects one source audio stream and has its own encoder settings
  (so the VOD track can, e.g., use a lower bitrate).
- **The order of the list defines the order of the output audio tracks.**
- "Add audio track" adds another entry; the arrows reorder, ✕ removes.

Typical setups:

| Target             | Audio tracks             | Result                              |
| ------------------ | ------------------------ | ----------------------------------- |
| Twitch             | Track 1, Track 2         | 1 = live audio, 2 = VOD audio       |
| YouTube / Kick     | Track 1                  | single mixed-down-stable live audio |
| Backup RTMP server | Track 2                  | only the clean game audio           |

For Twitch, position 1 is the live audio and position 2 is the VOD audio —
exactly what OBS's native "Twitch VOD track" produces.

## Compatibility notes

- **FLV/RTMP outputs with more than one audio track** are sent as Enhanced
  RTMP (like OBS does). This needs FFmpeg 8 or newer; the official bundles
  include it.
- Players and services that don't understand Enhanced RTMP (most legacy
  players) play **only the first audio track** and ignore the rest. That is
  exactly the Twitch semantics: live viewers hear track 1, the VOD gets
  track 2.
- Small timestamp drift between the tracks is passed through unchanged.
- Multi-track audio is **not** merged or downmixed anywhere in Restreamer —
  each selected track stays independent until the target encoder.

## Troubleshooting

- **"no streams available" in the log / publish rejected**: the Restreamer
  build doesn't understand Enhanced RTMP audio. Update to a build with this
  feature (FFmpeg 8 bundles).
- **The second track is missing at the target**: check that *both* tracks are
  selected in the "Streaming Audio Tracks" of OBS, that "Twitch VOD Track" is
  enabled (Method A), and that the target's "Audio tracks" list in Restreamer
  contains two entries.
- **The "Twitch VOD Track" option doesn't appear in OBS**: the stream service
  must be set to "Twitch" (with a custom server URL), and *Enable Custom
  Encoder Settings* must be checked. Simple output mode has no multi-track
  support at all.
- **Everything is merged into one track**: the sources share an OBS track —
  separate them in *Advanced Audio Properties* (see above).
