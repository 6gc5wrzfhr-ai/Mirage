# Sheridan multi-reference Mirage implementation

## Status

This is an implementation record for the `sheridan-multireference` branch. It is not a claim that multi-reference generation is already implemented.

## Verified upstream capabilities at the pinned sd.cpp revision

Mirage vendors `haplollc/stable-diffusion.cpp` at commit `9caad4d6c59f46615c21962fc349d67877c9213d`.

The pinned `stable-diffusion.h` already exposes these native inputs:

- Model context: `clip_vision_path`, `ip_adapter_path`, and `photo_maker_path`.
- Generation: `ip_adapter_image` plus `ip_adapter_strength`.
- PhotoMaker: `pm_params.id_images` plus `id_images_count`, `id_embed_path`, and `style_strength`.
- Generic model reference images: `ref_images`, `ref_images_count`, and `ref_image_args`.

The current Mirage C bridge does not forward these model paths or generation inputs. Its `mirage_gen_params` is prompt/sampler-only, and `mirage_ctx_create` does not set the optional vision/adapter/PhotoMaker paths.

## Model-specific constraints that must remain explicit

1. IP-Adapter Plus in the pinned sd.cpp documentation is for SD 1.5 and SDXL, and its generation API accepts one `ip_adapter_image` per call. It requires a CLIP-Vision encoder and adapter weights.
2. The pinned PhotoMaker documentation states PhotoMaker v1 is SDXL-only. Its C API accepts an array of identity images, so this is the available native multi-face-image route, but it requires an SDXL-compatible model and PhotoMaker weights.
3. `ref_images[]` is a generic model input, not a universal identity/style adapter. Its effect depends on the selected model architecture. It must not be described as generic multi-IP-Adapter conditioning.
4. Therefore, the current Sheridan SD 1.5 configuration cannot use PhotoMaker v1 as-is. Do not silently switch models or replace the working generator before measuring iPhone memory and performance.

## Implementation sequence

### A. Extend Mirage's C ABI

- Add optional CLIP-Vision, IP-Adapter, and PhotoMaker model paths to `mirage_model_paths`.
- Add a borrowed RGBA/RGB image-view type with explicit width, height, and channel count.
- Add arrays of identity images and generic reference images to `mirage_gen_params`, plus counts and the applicable strengths/arguments.
- Validate pointers, dimensions, channel counts, and array counts before calling sd.cpp.
- Map the new values to `sd_ctx_params_t` and `sd_img_gen_params_t`; keep all image buffers alive for the synchronous generation call.
- Preserve the existing prompt-only API behaviour when the new fields are absent.

### B. Extend Mirage's Swift facade

- Expose optional model URLs for CLIP-Vision, IP-Adapter, and PhotoMaker.
- Expose role-tagged reference inputs without implying that every role is natively supported by every model.
- Convert CGImage/UIImage inputs to the native pixel format with explicit ownership and no dangling buffers.
- Add validation errors for incompatible model/adapter combinations.

### C. Integrate Sheridan

- Pass every face reference to PhotoMaker only when an SDXL + PhotoMaker configuration is selected and validated.
- Pass a single selected image to the IP-Adapter path only where the model/adapter supports it; do not pretend that one image represents the full library.
- Pass multiple `ref_images` only for model families whose documentation confirms support and define what the references mean.
- Keep body, tattoo, clothing and mature-style role handling separate in Sheridan's data model. A future multi-adapter implementation is required if those roles must independently influence one image on SD 1.5.

### D. Required validation

- C ABI compile and symbol export checks for device and simulator slices.
- Swift API compile and unit tests.
- Backward-compatibility smoke test for prompt-only generation.
- Tests for zero, one, and multiple face/reference images; malformed images; and missing optional model files.
- An SDXL + PhotoMaker multi-face test before claiming PhotoMaker identity support.
- An SD 1.5 + IP-Adapter single-reference test before claiming IP-Adapter support.
- Measure model download size, peak memory, generation time, heat, and battery on the actual iPhone 15 Pro.
- Visual identity evaluation across varied poses, expressions, clothing and scenes.

## Immediate engineering decision

Implement the bridge in small, reviewable commits, starting with the optional model paths and typed image-array ABI. Build the Mirage XCFramework on GitHub Actions after the C++ and Swift layers agree. Keep Sheridan's current generator intact until the new framework compiles and a model-compatible test passes.
