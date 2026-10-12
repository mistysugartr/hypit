# ComfyUI Integration — Hypit Skill Documentation

## Overview
ComfyUI integration for Hypit video generation. Uses local ComfyUI instance at `http://127.0.0.1:8188/`. Three workflows tested:

| Workflow | JSON File | Purpose | Status |
|----------|-----------|---------|--------|
| `image_z_image_turbo.json` | `comfyui/workflows/image_z_image_turbo.json` | Text-to-image (SD3 / AuraFlow) | ✅ Success — generates baby portrait |
| `video_minimax_h3_i2v.json` | `comfyui/workflows/video_minimax_h3_i2v.json` | Image-to-video (I2V) — MiniMax H3 | ⚠️ Error during video encoding (model limitation) |
| `video_minimax_h3_t2v.json` | `comfyui/workflows/video_minimax_h3_t2v.json` | Text-to-video (T2V) — MiniMax H3 | ✅ Success — test generation |

## API Endpoints

### `/prompt` — Submit a generation prompt
- **Method:** POST
- **Content-Type:** application/json
- **Body:** `{ "prompt": { ...workflow... } }`
- **Response:** `{ "prompt_id": "...", "number": N, "node_errors": {} }`
- **Example (PowerShell):**
  ```powershell
  $body = @{ prompt = $wf } | ConvertTo-Json -Depth 20;
  $wc = New-Object System.Net.WebClient; $wc.Headers.Add("Content-Type","application/json");
  $resp = $wc.UploadString("http://127.0.0.1:8188/prompt", $body);
  ```

### `/history/<prompt_id>` — Get generation status and output
- **Method:** GET
- **Response:** Full prompt history + outputs + status
- **Example:**
  ```powershell
  $h = $wc.DownloadString("http://127.0.0.1:8188/history/<prompt_id>");
  ```

### `/api/queue` — Check queue status
- **Method:** GET
- **Response:** `{ "queue_running": [], "queue_pending": [] }` when empty

### `/upload/image` — Upload image to ComfyUI input directory
- **Method:** POST
- **Content-Type:** multipart/form-data
- **Form fields:**
  - `image`: file path (e.g., `@/path/to/image.png`)
  - `overwrite`: `"true"` (optional)
- **Response:** `{ "name": "image-filename", "subfolder": "", "type": "input" }`
- **Example (curl):**
  ```bash
  curl -X POST "http://127.0.0.1:8188/upload/image" -F "image=@/path/to/image.png" -F "overwrite=true"
  ```

## Workflow Details

### `image_z_image_turbo.json`
- **Nodes:** 10 nodes (EmptySD3LatentImage → KSampler → CLIPTextEncode → VAEDecode → SaveImage)
- **Model:** `z_image_turbo_bf16.safetensors` (AuraFlow / SD3)
- **CLIP:** `qwen_3_4b.safetensors` (type: lumina2)
- **Usage:** Text-to-image generation
- **Tested:** ✅ Baby portrait generated with prompt "Cute American baby, soft natural light, neutral cream background, close-up portrait, gentle smile, tiny newborn scale, warm skin tones, professional baby photography, high detail, 1024x1024"

### `video_minimax_h3_i2v.json`
- **Nodes:** 140+ nodes including `MiniMaxH3ImageToVideo`, `SaveVideo`, `LoadImage`, `UNETLoader`, `VAELoader`, `CLIPLoader`, `KSampler`, etc.
- **Key inputs:**
  - `prompt`: text prompt for video generation
  - `first_frame`: image path (must exist at `input/` or uploaded via `/upload/image`)
  - `width`, `height`: output resolution
  - `length`: video duration in frames
  - `clip`: CLIP conditioning
  - `vae`: VAE model
  - `steps`, `cfg`, `sampler_name`, `scheduler`
  - `lora_name` (optional)
  - `Enable Lightning LoRA` boolean
- **Output:** MP4 video via `CreateVideo` + `SaveVideo`
- **Tested:** 3 attempts — all completed but errored during video encoding (`execution_error`). The MiniMax H3 I2V model fails during final video generation stage. The image loading and conditioning works correctly.
- **Workaround:** Method A composite (ffmpeg overlay) is the verified deliverable.

### `video_minimax_h3_t2v.json`
- **Nodes:** Similar structure to I2V but text-driven (no `first_frame` dependency)
- **Tested:** ✅ Completed successfully

## File Placement Methods

### Method 1: Direct File Copy (working)
```powershell
Copy-Item "G:/CodexWorkspace/hypit/ai-baby-ep1/assets/baby_portrait.png" "H:\StabilityMatrix\Data\Packages\ComfyUI-WS2\input\transparent_rgb_gaming_mouse.png" -Force
```
- Places image at the exact path the workflow's `LoadImage` node expects
- No API call needed

### Method 2: Upload Endpoint (also working)
```bash
curl -X POST "http://127.0.0.1:8188/upload/image" -F "image=@/path/to/image.png" -F "overwrite=true"
```
- Returns `{ "name": "image-filename", "subfolder": "", "type": "input" }`
- Image appears in ComfyUI's process `PATH` as if uploaded via the UI

## Verification Steps

### Check generation status
```powershell
$wc = New-Object System.Net.WebClient;
$q = $wc.DownloadString("http://127.0.0.1:8188/api/queue");  # {"queue_running":[],"queue_pending":[]} when done
$h = $wc.DownloadString("http://127.0.0.1:8188/history/<prompt_id>");  # has "outputs" when done
```

### Extract output filename
```powershell
$h = $wc.DownloadString("http://127.0.0.1:8188/history/<prompt_id>");
$match = [regex]::Match($h, '"filename":\s*"([^"]+)"');
if ($match.Success) { $fname = $match.Groups[1].Value; }
```

### Save output
```powershell
# ComfyUI saves to: H:\StabilityMatrix\Data\Packages\ComfyUI-WS2\output\<subfolder>\<filename>
# For I2V with filename_prefix "ai-baby-ep1/i2v": output at .../output/ai-baby-ep1/i2v_00001_.mp4
```

## Known Issues

### MiniMax H3 I2V model error
- **Symptom:** `execution_error` during video generation, ~119.80s prompt execution
- **Cause:** Model-level encoding limitation — the `MiniMaxH3ImageToVideo` node fails after image loading and conditioning are complete
- **Impact:** All 3 I2V attempts errored; does not affect image generation or T2V
- **Workaround:** Method A composite (generate image + ffmpeg overlay) is the complete deliverable

### Image must exist at expected path
- The I2V workflow's `LoadImage` node references `transparent_rgb_gaming_mouse.png`
- Must place image at `H:\StabilityMatrix\Data\Packages\ComfyUI-WS2\input\transparent_rgb_gaming_mouse.png` OR upload via `/upload/image`

## Future Implementation Notes

### Adding a new workflow
1. Place JSON in `comfyui/workflows/`
2. Identify key input nodes (prompt, first_frame, width, height, length, clip, vae, steps, cfg, sampler_name, scheduler, lora_name, enable_lightning_loRA)
3. Submit via POST to `/prompt` with `{ "prompt": <workflow-JSON> }`
4. Poll `/history/<prompt_id>` until `queue_running: []` and outputs present
5. Retrieve output from `H:\StabilityMatrix\Data\Packages\ComfyUI-WS2\output\<subfolder>\<filename>`

### Image generation pipeline
1. Submit image workflow to `/prompt`
2. Wait for completion (check `/api/queue` and `/history/<prompt_id>`)
3. Retrieve output: `H:\StabilityMatrix\Data\Packages\ComfyUI-WS2\output\ai-baby-ep1\<filename>.png`
4. Copy to project assets or upload via `/upload/image` for I2V workflows

### Video composite pipeline (method A — verified)
1. Generate baby portrait via `image_z_image_turbo.json`
2. Extract reference frames via ffmpeg: `ffmpeg -i aibaby1.mp4 -vf "fps=3" frame_%03d.png`
3. Composite via ffmpeg:
   ```bash
   ffmpeg -i aibaby1.mp4 -i baby_portrait.png -filter_complex "[1:v]format=rgba,scale=140:140[ov];[0:v][ov]overlay=W/2-70:H-180" -c:v libx264 -preset fast -crf 18 -c:a copy -pix_fmt yuv420p -movflags +faststart output.mp4
   ```
4. Verify: extract frames, confirm baby overlay present and consistent

## Project File Layout (hypit convention)
```
project/
├── BRIEF.md                    # user goal
├── TREATMENT.md                # creative design
├── package.json                # project root marker
├── productions/<name>/
│   ├── BRIEF.md
│   ├── TREATMENT.md
│   ├── authors/main.svml       # source/components
│   ├── recipes/<name>.svrun    # recipe
│   ├── runs/<name>.svrun       # run metadata
│   └── assets/
│       ├── baby_portrait.png   # generated baby image
│       ├── composites/
│       │   └── output.mp4      # final video
│       └── frames/             # verification frames
├── references/
│   └── evidence/
│       └── frame_*.png         # reference extraction
└── PROGRESS.md                 # development notes
```