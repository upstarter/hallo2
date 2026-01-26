# Hallo2 - Claude Instructions

## CRITICAL: Always Use inference_long.py

**ALWAYS** use `scripts/inference_long.py` for ALL inference, regardless of audio duration.

```bash
python scripts/inference_long.py -c configs/inference/your_config.yaml
```

**NEVER** use `scripts/inference.py` - it lacks proper segment stitching and temporal consistency.

## Image Preparation

1. **Source image MUST have visible, open eyes** - squinting/closed eyes = artifacts
2. **Square crop required** - 512x512 recommended
3. **Face should be 50-70% of frame**
4. **Upscale low-res images first** with Real-ESRGAN:
   ```bash
   realesrgan-ncnn-vulkan -i input.jpg -o upscaled.png -n realesrgan-x4plus
   ```

## Audio Preparation

```bash
# Convert to 16kHz mono WAV
ffmpeg -i input.mp3 -ar 16000 -ac 1 output.wav

# Extract segment (e.g., 1.5s starting at 15s where lyrics begin)
ffmpeg -i input.wav -ss 15 -t 1.5 -ar 16000 -ac 1 segment.wav
```

## Quality Settings

| Parameter | Recommended | Notes |
|-----------|-------------|-------|
| `inference_steps` | 50 | Higher = better quality, slower |
| `cfg_scale` | 3.0 | 2.5-3.5 range |
| `lip_weight` | 1.2 | Boost for music/singing |
| `pose_weight` | 0.8 | Head movement |

## Venv

```bash
source /mnt/cache/venvs/hallo2/bin/activate
```

## Common Issues

- **Pixelated eyes**: Source image eyes not visible/open
- **OOM**: Close DaVinci Resolve and other GPU apps first
- **Event loop errors**: Fixed in our fork (attn_implementation="eager")
