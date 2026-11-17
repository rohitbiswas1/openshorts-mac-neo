# Run OpenShorts on a MacBook Neo

This profile is for an Apple Silicon Mac with 8 GB of unified memory. OpenShorts still needs local CPU time for downloading, transcription, face detection, reframing, and FFmpeg encoding. A Gemini key moves the *clip selection* language-model calls to Google; fal.ai and ElevenLabs keys move their respective generation features to those services. Keys do not move the entire clip pipeline off the Mac.

## What each stage does

| Stage | Where it runs with this profile | Main Mac cost |
| --- | --- | --- |
| Upload or URL download and source probing | Mac | Disk and network |
| Audio transcription | Mac, faster-whisper `base` on CPU with int8 | CPU and memory |
| Pick moments from the transcript | Gemini API | Small request; API quota/cost |
| Scene and subject detection | Mac, PySceneDetect and MediaPipe; YOLO loads only as a fallback | CPU |
| Crop, captions, encode | Mac, FFmpeg with two threads, one clip worker, 720 × 1280 output, and the `veryfast` x264 preset | CPU and disk |
| Timeline render, if used | Mac, Remotion/Chromium | Memory and CPU |
| AI Shorts and voiceover, if used | fal.ai / ElevenLabs | API quota/cost |

The Mac profile limits the backend to one job and one clip worker. It caps high frame rate sources at about 30 fps before the later passes, and renders vertical clips at 720 × 1280 instead of the default 1080 × 1920. That is about 56% fewer output pixels per frame. The `veryfast` x264 preset reduces encoder work but can make files larger. The editor stays available. These choices reduce CPU work and peak load, with some possible loss of transcription, scene-boundary, and video detail. Total battery savings depend on the source and the Mac; they need a real-device measurement.

## Start

1. Install Docker Desktop for Apple Silicon. Give its Linux VM about **5 GB of memory** if your Mac has 8 GB total; keep enough memory for macOS and close other heavy apps while processing. Make sure you have **at least 25 GB of free disk** for images, model cache, and videos. The first build/download is large and can take a while. Keep the Mac plugged in for long jobs.
2. Open Terminal in this modified OpenShorts checkout, then run:

   ```bash
   docker compose -f compose.mac.yaml up --build -d
   ```

3. Open <http://localhost:5175>. In **Settings**, add a Gemini API key. Add fal.ai, ElevenLabs, or Upload-Post keys only if you use those features. Start with a short 720p source video to check the complete flow.

The frontend and API are bound to `127.0.0.1`; other devices cannot connect to them over the network. The renderer stays on the private Compose network.

## Use less of the Mac

- Process one source at a time. Prefer short 720p sources, especially while testing; a 4K or hour-long file still needs substantial local decoding and disk space.
- To restore the normal output detail, set `DELIVERY_MIN_WIDTH` to `1080`, `DELIVERY_MAX_WIDTH` to `0`, and remove `X264_PRESET` from `compose.mac.yaml`. Set `SELF_HOST_MAX_FPS` to `0` to keep the source frame rate. Recreate the backend after changes.
- Gemini is the lightest choice for moment selection on this Mac. Running Ollama locally would add another model to the same 8 GB memory pool.
- If you only need automatic clipping, start just the backend and frontend:

  ```bash
  docker compose -f compose.mac.yaml up --build -d backend frontend
  ```

  Timeline rendering through the Remotion service requires the renderer too. Start it later with `docker compose -f compose.mac.yaml up -d renderer`.
- If `base` makes too many transcript errors, change `WHISPER_MODEL` to `small` in `compose.mac.yaml` and recreate the backend with `docker compose -f compose.mac.yaml up -d --force-recreate backend`. This uses more CPU and memory. Change it back to `base` if the Mac struggles.
- If scene cuts are poor, set `SCENE_ENGINE` to `transnetv2`. This loads a Torch model and uses more memory; revert to `pyscenedetect` for the lighter profile.

## Check and stop

```bash
docker compose -f compose.mac.yaml ps
docker compose -f compose.mac.yaml logs -f backend
docker compose -f compose.mac.yaml down
```

`down` stops the containers while keeping downloaded models, uploads, and output in Docker named volumes. Finished jobs are still subject to the app's normal 24-hour retention and disk caps. Download clips you want to keep. **Do not use `down -v`** unless you intend to delete those volumes. Docker Desktop itself can be quit after you stop OpenShorts.

If `localhost:5175` does not open, check `docker compose -f compose.mac.yaml ps` and the frontend/backend logs. On the first run, allow time for model downloads. If Docker reports an out-of-memory kill (exit 137), stop the renderer, close other applications, and try a shorter source.
