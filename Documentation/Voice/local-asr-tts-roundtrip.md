# Local ASR, TTS, and Voice Round-Trip on Jibo (Post-Cloud)

> This document describes the first confirmed working voice interaction on a Jibo robot after official cloud services were discontinued.

---

## Summary

Short version: Jibo can still have a full conversation loop locally.

We now have:

* Speech → text (STT) working locally
* Text → speech (TTS) working locally
* A working loop where Jibo hears something and responds

This is all happening without the original cloud services.

---

## Key Findings

Here’s what we now know for sure:

* Wake word detection (`hey jibo`) still works locally
* Speaker ID is still running locally (even if it rejects us 😄)
* `jibo-asr-service` can be started and controlled manually
* ASR (speech recognition) is exposed over HTTP on port `8088`
* TTS (speech output) is exposed over HTTP on port `8089`

### ASR Endpoints

Confirmed working endpoints:

* `/asr_simple_interface`
* `/audio_source`
* `/asr_control`
* `/status`

### WebSocket Outputs

ASR results are streamed over WebSockets:

* `ws://<jibo-ip>:8088/port`
* `ws://<jibo-ip>:8088/simple_port`

### Example STT Start Payload

```json
{
  "command": "start",
  "task_id": "DEBUG:task3",
  "audio_source_id": "alsa1",
  "hotphrase": "none",
  "speech_to_text": true,
  "request_id": "stt_start3"
}
```

---

## What’s Actually Happening (Architecture)

Here’s the real flow in plain English:

1. We send a request to Jibo to start listening
2. Jibo captures audio from its mic (ALSA)
3. The ASR engine processes it
4. Results come back over WebSocket
5. Our app reads the transcript
6. Our app decides what to say
7. We send that to Jibo’s TTS
8. Jibo speaks

Visual version:

```
HTTP POST (/asr_simple_interface)
        ↓
ASR service captures audio
        ↓
Speech recognition runs locally
        ↓
WebSocket emits events
        ↓
External app receives transcript
        ↓
External logic decides response
        ↓
HTTP POST (/tts_speak)
        ↓
Jibo talks
```

---

## Example WebSocket Output

Here’s a trimmed real example of a final result:

```json
{
  "event_type": "speech_to_text_final",
  "task_id": "DEBUG:task3",
  "utterances": [
    {
      "utterance": "what time is it",
      "score": 975.9
    }
  ]
}
```

You’ll also see:

* `speech_to_text_incremental` (partial results)
* `end_of_speech`
* `hotphrase` (for "hey jibo")

---

## Demo Flow (How to Reproduce)

This is the important part.

### 1. Make sure you are in `int-developer` mode and ASR service is running

From ssh:

```
/usr/local/bin/jibo-asr-service -c /usr/local/etc/jibo-asr-service.json
```

---

### 2. Connect to WebSocket

```
ws://<jibo-ip>:8088/simple_port
```

---

### 3. Start an STT task

POST to:

```
http://<jibo-ip>:8088/asr_simple_interface
```

With:

```json
{
  "command": "start",
  "task_id": "DEBUG:task3",
  "audio_source_id": "alsa1",
  "hotphrase": "none",
  "speech_to_text": true
}
```

---

### 4. Speak to Jibo

Say something like:

> “what time is it”

---

### 5. Wait for final transcript

Watch for:

```
event_type: speech_to_text_final
```

---

### 6. Send response to TTS

POST to:

```
http://<jibo-ip>:8089/tts_speak
```

With something like:

```json
{
  "text": "It is demo time."
}
```

---

### 7. Jibo speaks 🎉

---

## Known Behaviors / Quirks

Some things we’ve seen so far:

* WebSocket connections can drop → reconnect logic helps
* Incremental results can be messy or duplicated
* Multiple transcript guesses can show up
* Wake word (`task0`) runs alongside your custom task
* Saying “hey jibo” during a manual STT session can interfere
* Speaker ID often rejects (but doesn’t block STT)

---

## Corrections to Previous Assumptions

Some things we (and others) thought before that are now clearly wrong or incomplete:

* “ASR is dead without cloud” → **Not true in developer mode**
* “Only wake word works locally” → **Incomplete**
* “No way to get transcripts” → **False (WebSocket output exists)**
* “Jibo can’t answer questions anymore” → **Also false now 🙂**

---

## What This Means

This is a big deal:

* Jibo’s core voice pipeline is still there
* The cloud was orchestration, not the whole system
* We can now rebuild the “brain” externally

---

## Next Steps

Where this naturally goes next:

* Hook wake word → automatically trigger STT
* Figure out how this behaves in “normal mode”
* See if Jibo tries to initiate outbound connections (old cloud flow)
* Intercept or replace those endpoints locally
* Build a simple always-on bridge service:

  * Wake word → STT → AI → TTS

---

## Final Thought

We didn’t just poke at endpoints here.

We proved Jibo can:

* hear
* understand
* and respond again

That’s a pretty great place to be.
