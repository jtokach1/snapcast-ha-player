Replace the contents of:
custom_components/snapcast_player/media_player.py



# Snapcast / Music Assistant No-Audio Fix

## Summary

Music Assistant appeared to be playing normally, and Snapserver/Snapclient logs looked healthy, but there was no sound from either of the Home Assistant bridge players.

Music Assistant reported:

```text
PlayerCommandFailed
```

The problem was not Music Assistant, Snapserver, or the Snapclients. It was a PCM format mismatch in the custom Home Assistant integration:

```text
/config/custom_components/snapcast_player/media_player.py
```

## Symptoms

- Music Assistant showed the source as playing.
- Other Music Assistant players worked normally.
- Music Assistant displayed `PlayerCommandFailed`.
- Snapserver remained online and clients stayed connected.
- No audio reached Snapcast from the Home Assistant bridge players.
- Home Assistant logs showed the FFmpeg process exiting, followed by errors such as:

```text
ProcessLookupError
PlayerCommandFailed
```

After diagnostic logging was added, Home Assistant showed FFmpeg exiting with nonzero return codes.

## Root Cause

The custom `snapcast_player` integration launched FFmpeg with conflicting raw PCM settings:

```python
"-f",
"u16le",
"-acodec",
"pcm_s16le",
```

These settings do not agree:

- `u16le` means unsigned 16-bit little-endian PCM.
- `pcm_s16le` means signed 16-bit little-endian PCM.

FFmpeg exited before it could send audio to the Snapserver TCP input. The integration then tried to stop or pause an already-dead FFmpeg process, which produced the misleading `ProcessLookupError` and surfaced in Music Assistant as `PlayerCommandFailed`.

## Fix

Edit:

```text
/config/custom_components/snapcast_player/media_player.py
```

Find:

```python
format_args = [
    "-f",
    "u16le",
    "-acodec",
    "pcm_s16le",
    "-ac",
    "2",
    "-ar",
    "48000",
]
```

Change only `u16le` to `s16le`:

```python
format_args = [
    "-f",
    "s16le",
    "-acodec",
    "pcm_s16le",
    "-ac",
    "2",
    "-ar",
    "48000",
]
```

Then restart Home Assistant.

## Result

After the change:

- Music Assistant could play to Players.
- Audio reached Snapserver correctly.
- Snapclients produced synchronized audio again.
- `PlayerCommandFailed` stopped occurring for normal playback.
