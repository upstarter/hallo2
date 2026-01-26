# Hallo2 Local Usage Guide

Quick reference for generating lip-sync talking head videos.

## Quick Start

```bash
cd /mnt/dev/ai/hallo2
source /mnt/cache/venvs/hallo2/bin/activate

# Run inference
python scripts/inference_long.py -c configs/inference/your_config.yaml
```

## Source Image Requirements (CRITICAL)

| Requirement | Why |
|-------------|-----|
| **Square crop** | Model expects 512x512 |
| **Face = 50-70% of frame** | Too small = poor detail, too large = clipping |
| **Eyes visible and open** | Can't generate realistic eyes from squinting |
| **Forward-facing** | Rotation < 30 degrees |
| **Good resolution** | Upscale with Real-ESRGAN first if low-res |

### Image Preparation Workflow

```bash
# 1. If image is low-res, upscale first with Real-ESRGAN
realesrgan-ncnn-vulkan -i input.jpg -o upscaled.png -n realesrgan-x4plus

# 2. Create square crop centered on face
convert upscaled.png \
  -gravity center -crop 1:1 +repage \
  -resize 512x512 \
  output.png
```

## Audio Requirements

- **Format:** WAV only (16kHz sample rate, mono)
- **Language:** English (training data limitation)
- **Clear vocals:** Background music OK, but vocals must be clear

### Audio Conversion

```bash
# Convert any audio to proper format
ffmpeg -i input.mp3 -ar 16000 -ac 1 output.wav

# Extract segment (e.g., 1.5 seconds starting at 15s)
ffmpeg -i input.wav -ss 15 -t 1.5 -ar 16000 -ac 1 segment.wav
```

## Config File Template

```yaml
source_image: /path/to/portrait.png
driving_audio: /path/to/audio.wav

weight_dtype: fp16

data:
  n_motion_frames: 2
  n_sample_frames: 16
  source_image:
    width: 512
    height: 512
  driving_audio:
    sample_rate: 16000
  export_video:
    fps: 25

# Quality settings (higher = better but slower)
inference_steps: 50  # 40-50 recommended
cfg_scale: 3.0       # 2.5-3.5 range

use_mask: true
mask_rate: 0.25
use_cut: true

audio_ckpt_dir: pretrained_models/hallo2/hallo2
save_path: ./output/my_video/
cache_path: ./.cache

base_model_path: ./pretrained_models/hallo2/stable-diffusion-v1-5
motion_module_path: ./pretrained_models/hallo2/motion_module/mm_sd_v15_v2.ckpt

face_analysis:
  model_path: ./pretrained_models/face_analysis

wav2vec:
  model_path: ./pretrained_models/hallo2/wav2vec/wav2vec2-base-960h
  features: all

audio_separator:
  model_path: ./pretrained_models/hallo2/audio_separator/Kim_Vocal_2.onnx

vae:
  model_path: ./pretrained_models/hallo2/sd-vae-ft-mse

# Animation weights
face_expand_ratio: 1.1  # 1.0-1.2, smaller = tighter face crop
pose_weight: 0.8        # Head movement
face_weight: 1.0        # Facial expressions
lip_weight: 1.2         # Lip sync accuracy (boost for music)

unet_additional_kwargs:
  use_inflated_groupnorm: true
  unet_use_cross_frame_attention: false
  unet_use_temporal_attention: false
  use_motion_module: true
  use_audio_module: true
  motion_module_resolutions: [1, 2, 4, 8]
  motion_module_mid_block: true
  motion_module_decoder_only: false
  motion_module_type: Vanilla
  motion_module_kwargs:
    num_attention_heads: 8
    num_transformer_block: 1
    attention_block_types: [Temporal_Self, Temporal_Self]
    temporal_position_encoding: true
    temporal_position_encoding_max_len: 32
    temporal_attention_dim_div: 1
  audio_attention_dim: 768
  stack_enable_blocks_name: ["up", "down", "mid"]
  stack_enable_blocks_depth: [0, 1, 2, 3]

enable_zero_snr: true

noise_scheduler_kwargs:
  beta_start: 0.00085
  beta_end: 0.012
  beta_schedule: "linear"
  clip_sample: false
  steps_offset: 1
  prediction_type: "v_prediction"
  rescale_betas_zero_snr: true
  timestep_spacing: "trailing"

sampler: DDIM
```

## Tuning Parameters

| Parameter | Effect | Recommended |
|-----------|--------|-------------|
| `inference_steps` | Quality vs speed | 40-50 |
| `cfg_scale` | Prompt adherence | 2.5-3.5 |
| `lip_weight` | Lip sync accuracy | 1.0-1.5 |
| `pose_weight` | Head movement amount | 0.5-1.0 |
| `face_expand_ratio` | Face crop tightness | 1.0-1.2 |

## Troubleshooting

### Pixelated eyes/artifacts
- Source image eyes must be **visible and open**
- Upscale source image with Real-ESRGAN first
- Increase `inference_steps` to 50

### Poor lip sync
- Increase `lip_weight` to 1.2-1.5
- Ensure audio is clear vocals (separate music if needed)
- Use WAV format at 16kHz

### Out of memory
- Close other GPU applications (Resolve, etc.)
- Reduce `inference_steps` to 30
- RTX 5080 16GB can handle 512x512 comfortably

### Event loop errors
- Fixed in our fork - uses `attn_implementation="eager"` for wav2vec

## VRAM Usage

- Inference: ~12GB for 512x512
- Ensure no other GPU apps running (DaVinci Resolve uses 7GB+)

## Output Location

Videos saved to: `{save_path}/{source_image_name}/merge_video.mp4`
