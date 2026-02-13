---
name: "scaled-surrogate-gradient-codec-aware-learning"
description: "Build end-to-end video processing pipelines that train learned downsamplers/upsamplers through real non-differentiable codecs using data-driven surrogate gradients. Trigger phrases: 'codec-aware downsampling', 'train through non-differentiable codec', 'surrogate gradient video pipeline', 'learned resampling ABR streaming', 'end-to-end codec training', 'optimize downsampler with real encoder'"
---

# SCALED: Surrogate-Gradient Codec-Aware Learning of Downsampling

This skill enables Claude to build video processing pipelines where a learned downsampler (and optionally upsampler) is trained end-to-end through a real, non-differentiable video codec (e.g., VVC/H.266, HEVC/H.265, AVC/H.264) by computing data-driven surrogate gradients from actual compression artifacts. Instead of replacing the codec with a differentiable proxy network that only approximates encoding behavior, SCALED keeps the real codec in the training loop and estimates gradients empirically from observed compression errors, closing the gap between training-time and deployment-time performance.

## When to Use

- When a user wants to **train a neural downsampler or pre-processor** that feeds into a standard video codec (x265, x266, VVenC, FFmpeg-based) and needs the downsampler to be codec-aware rather than codec-agnostic.
- When building an **Adaptive Bitrate (ABR) streaming pipeline** that jointly optimizes resolution selection, downsampling, encoding, and super-resolution upsampling for rate-distortion performance.
- When the user asks how to **backpropagate through a non-differentiable encoder** without building a differentiable proxy codec or neural codec surrogate.
- When optimizing a **learned image/video resampling network** (downsampler + upsampler pair) where the bottleneck is a traditional codec and the objective is to minimize BD-BR across multiple quality points.
- When the user wants to **compare proxy-codec vs. surrogate-gradient** training strategies for compression-aware neural networks.
- When implementing **straight-through estimator (STE) variants** or surrogate gradient methods for any non-differentiable black-box module in a video/image pipeline.

## Key Technique

### The Problem with Proxy Codecs

In a standard ABR streaming pipeline, high-resolution content is downsampled, encoded at a target bitrate, transmitted, decoded, and upsampled on the client. When using neural networks for the down/upsampling stages, end-to-end gradient-based training requires differentiating through the codec — but standard codecs (VVC, HEVC) are not differentiable. Prior work replaces the codec with a differentiable proxy: either a neural network trained to mimic the codec, or a hybrid coder with soft quantization. The proxy introduces a train-test mismatch: the network optimizes for the proxy's compression behavior, not the real codec's, and this gap degrades deployment performance.

### Data-Driven Surrogate Gradients from Real Compression Errors

SCALED eliminates the proxy by running the actual codec in the forward pass and estimating gradients for backpropagation empirically. The core idea: for a batch of downsampled frames, encode them with the real codec, decode, and compute the compression error (the difference between the codec input and codec output). This compression error is a function of the downsampled content — changing the downsampler's output changes what the codec sees and therefore changes the compression artifacts. By treating the codec as a black box and measuring how its output responds to perturbations of its input (or by using the compression residual directly as a gradient signal), SCALED constructs a surrogate gradient that points in a direction reducing end-to-end distortion under actual codec behavior.

Concretely, the surrogate gradient replaces the true (unavailable) Jacobian of the codec with an empirical estimate derived from the observed compression residual. This is analogous to the straight-through estimator used in quantization-aware training, but instead of assuming the gradient passes through unchanged, the surrogate is shaped by real codec statistics. The result is a 5.19% BD-BR (PSNR) improvement over codec-agnostic baselines — consistent across the full rate-distortion convex hull and multiple downsampling ratios.

## Step-by-Step Workflow

1. **Define the pipeline architecture.** Implement three modules: (a) a learnable downsampler network (e.g., a small CNN or residual network that outputs a lower-resolution frame), (b) a fixed non-differentiable codec invoked via subprocess (FFmpeg/VVenC/x265), and (c) a learnable upsampler network (e.g., a lightweight super-resolution CNN). Wire them sequentially: input frame -> downsampler -> codec encode/decode -> upsampler -> reconstructed frame.

2. **Implement the codec wrapper as a non-differentiable forward function.** Write a Python function that takes a downsampled tensor, saves it to a temporary raw/YUV file, invokes the codec encoder+decoder via subprocess at a specified QP or bitrate, reads back the decoded output, and returns it as a tensor. This function has no backward — it is a black box.

3. **Compute the compression residual.** After the codec forward pass, calculate `r = x_decoded - x_downsampled` (the difference between what went into the codec and what came out). This residual encodes the codec's compression behavior for the specific content and quality setting.

4. **Construct the surrogate gradient for the codec.** During backpropagation, when the autograd engine reaches the codec boundary, substitute the true (unknown) codec Jacobian with a surrogate: use the compression residual `r` to define a local linear approximation. In practice, implement this with a custom `torch.autograd.Function` whose `forward()` calls the real codec and whose `backward()` returns a gradient derived from the compression error — for instance, passing the upstream gradient through scaled by `(1 + alpha * r)` or using a finite-difference estimate obtained by encoding slightly perturbed inputs.

5. **Define the end-to-end loss function.** Use a weighted rate-distortion loss: `L = D(x_original, x_reconstructed) + lambda * R`, where `D` is a distortion metric (MSE, SSIM, or LPIPS between the original high-res frame and the final upsampled output) and `R` is the bitrate produced by the codec (read from encoder logs). Since `R` is also non-differentiable, treat it as a constant for a given QP or use a rate estimator.

6. **Train across multiple operating points.** For each training iteration, randomly sample a QP value (or bitrate target) from the range of interest. This ensures the downsampler learns to produce content that compresses well across the full R-D curve, not just a single quality point.

7. **Implement the training loop.** For each batch: run the downsampler (differentiable), invoke the codec wrapper (non-differentiable forward, surrogate backward), run the upsampler (differentiable), compute the loss against the original high-res frame, and backpropagate through the surrogate. Use Adam or AdamW with a learning rate around 1e-4.

8. **Evaluate with BD-BR metrics.** After training, encode a test set at multiple QP values using the real codec with the learned downsampler, decode, upsample, and compute PSNR/SSIM at each point. Calculate BD-BR (Bjontegaard Delta Bit Rate) against a baseline (e.g., bicubic downsampling) to measure the percentage bitrate savings at equivalent quality.

9. **Validate surrogate gradient quality.** Periodically compare the surrogate gradient direction against a finite-difference numerical gradient (encode at input, encode at input+epsilon, measure output change). If the cosine similarity between surrogate and numerical gradients drops below 0.5, increase perturbation diversity or adjust the surrogate scaling factor.

10. **Deploy the trained downsampler.** Export the learned downsampler as ONNX or TorchScript. Integrate it into the ABR encoding pipeline before the real codec. The upsampler is deployed on the client side. No proxy codec is needed at inference — the real codec is used throughout.

## Concrete Examples

**Example 1: Training a codec-aware downsampler with FFmpeg x265**

User: "I want to train a neural downsampler that produces 540p frames from 1080p input, optimized for x265 encoding. The downsampled frames should compress better than bicubic at equivalent quality."

Approach:
1. Build a 4-layer residual CNN downsampler (1080p -> 540p) and a corresponding ESPCN upsampler (540p -> 1080p).
2. Create the codec wrapper:

```python
import subprocess, torch, tempfile, numpy as np

class CodecForward(torch.autograd.Function):
    @staticmethod
    def forward(ctx, x_down, qp):
        # x_down: [B, C, H, W] tensor, already on CPU as uint8-range float
        ctx.save_for_backward(x_down)
        ctx.qp = qp
        decoded_frames = []
        for i in range(x_down.shape[0]):
            frame = x_down[i].detach().cpu().numpy()
            with tempfile.NamedTemporaryFile(suffix='.yuv') as fin, \
                 tempfile.NamedTemporaryFile(suffix='.yuv') as fout:
                # Write raw YUV frame
                write_yuv420(fin.name, frame)
                # Encode + decode with x265 via FFmpeg
                subprocess.run([
                    'ffmpeg', '-y', '-f', 'rawvideo',
                    '-pix_fmt', 'yuv420p',
                    '-s', f'{frame.shape[2]}x{frame.shape[1]}',
                    '-i', fin.name,
                    '-c:v', 'libx265', '-qp', str(qp),
                    '-f', 'rawvideo', fout.name
                ], capture_output=True)
                decoded = read_yuv420(fout.name, frame.shape[1], frame.shape[2])
                decoded_frames.append(torch.from_numpy(decoded))
        return torch.stack(decoded_frames).to(x_down.device)

    @staticmethod
    def backward(ctx, grad_output):
        x_down, = ctx.saved_tensors
        # Surrogate gradient: pass gradient through, scaled by
        # local compression sensitivity
        with torch.no_grad():
            x_decoded = CodecForward.apply(x_down, ctx.qp)
            residual = x_decoded - x_down
            # Surrogate: attenuate gradient where codec distorts heavily
            surrogate_scale = 1.0 / (1.0 + residual.abs())
        return grad_output * surrogate_scale, None
```

3. Train for 50 epochs on a dataset of 1080p video frames, sampling QP uniformly from {22, 27, 32, 37}.
4. Evaluate BD-BR: encode test sequences with learned downsampler + x265 vs. bicubic + x265 at QP {22, 27, 32, 37}, measure PSNR, compute BD-BR savings.

Output: A trained downsampler that saves ~5% BD-BR over bicubic resampling when used with x265.

---

**Example 2: Comparing surrogate gradient vs. proxy codec training**

User: "I've been training my downsampler with a neural proxy codec. How do I switch to surrogate gradients with the real codec and compare results?"

Approach:
1. Keep the existing downsampler and upsampler architectures unchanged.
2. Replace the proxy codec module with the real codec wrapper using `torch.autograd.Function` as shown above.
3. Implement two training configurations:

```python
# Config A: Proxy codec (existing approach)
pipeline_proxy = nn.Sequential(downsampler, proxy_codec, upsampler)

# Config B: Real codec + surrogate gradient (SCALED approach)
class SCALEDPipeline(nn.Module):
    def __init__(self, downsampler, upsampler, codec_cmd, qp):
        super().__init__()
        self.down = downsampler
        self.up = upsampler
        self.codec_cmd = codec_cmd
        self.qp = qp

    def forward(self, x_hr):
        x_down = self.down(x_hr)
        x_coded = CodecForward.apply(x_down, self.qp)
        x_recon = self.up(x_coded)
        return x_recon
```

4. Train both for the same number of epochs and compare:
   - Training loss convergence curves
   - Test BD-BR (PSNR) against bicubic baseline
   - Per-sequence R-D curves at QP {22, 27, 32, 37}
5. The surrogate gradient variant should show better deployment performance because it optimizes directly for the real codec's behavior.

Output: Side-by-side BD-BR comparison table showing the surrogate gradient approach outperforming the proxy codec approach, especially at extreme bitrates where proxy approximation errors are largest.

---

**Example 3: Integrating SCALED into an ABR streaming server**

User: "I have a live ABR encoding server using FFmpeg. I want to add a learned pre-processing step before encoding that adapts the downsampling to the codec."

Approach:
1. Train the downsampler offline using the SCALED surrogate gradient method against the target codec (e.g., libx264 at CRF values used in production).
2. Export the trained model:

```python
traced = torch.jit.trace(downsampler, torch.randn(1, 3, 1080, 1920))
traced.save("scaled_downsampler_x264.pt")
```

3. Create a pre-processing service that loads the model and processes frames before they reach FFmpeg:

```python
import torch, sys

model = torch.jit.load("scaled_downsampler_x264.pt")
model.eval()

def preprocess_frame(frame_rgb: np.ndarray) -> np.ndarray:
    """Apply learned downsampling before codec encoding."""
    tensor = torch.from_numpy(frame_rgb).permute(2,0,1).unsqueeze(0).float() / 255.0
    with torch.no_grad():
        downsampled = model(tensor)
    return (downsampled.squeeze(0).permute(1,2,0) * 255).byte().numpy()
```

4. Pipe the pre-processed frames into FFmpeg for encoding at each ABR ladder rung.
5. On the client side, deploy the corresponding upsampler as a post-processing filter after decoding.

Output: An ABR pipeline where the downsampler is optimized for the actual production codec, yielding ~5% bitrate savings at equivalent visual quality across all ladder rungs.

## Best Practices

- **Do:** Run the real codec at the exact settings (profile, preset, QP range) you will use in production. The surrogate gradient's value comes from matching training conditions to deployment conditions exactly.
- **Do:** Sample QP/CRF values randomly during training to cover the full R-D curve. A model trained at a single quality point will not generalize well across the bitrate ladder.
- **Do:** Normalize the compression residual before using it in the surrogate gradient. Raw residuals can vary by orders of magnitude across QP values, destabilizing training.
- **Do:** Use small batch sizes (4-8 frames) to keep codec invocation overhead manageable, and accumulate gradients over multiple mini-batches if needed.
- **Avoid:** Using the surrogate gradient with a codec running in multi-threaded mode where frame-level non-determinism could corrupt gradient estimates. Use single-threaded or deterministic encoding settings during training.
- **Avoid:** Training with only MSE loss. Supplement with perceptual losses (SSIM, LPIPS) since codec artifacts have perceptual structure that MSE alone does not capture well.
- **Avoid:** Skipping the finite-difference gradient validation step. If your surrogate diverges from the numerical gradient, the network may converge to a suboptimal solution without any obvious signal in the training loss.

## Error Handling

- **Codec subprocess crashes:** Wrap the codec invocation in try/except. If a frame fails to encode (corrupt input, codec bug), skip that sample and log it. Do not let a single failed encode crash the training loop.
- **Gradient explosion from large residuals:** At low QP (high quality), compression residuals are tiny; at high QP (low quality), they can be large. Clamp the surrogate scaling factor to a reasonable range (e.g., [0.1, 10.0]) to prevent gradient explosion or vanishing.
- **Bitrate readout failures:** Parsing codec output for bitrate can fail if the log format changes between codec versions. Write a robust parser with fallback regex patterns, or read the output file size directly.
- **Memory overflow with large resolutions:** 4K frames encoded per-sample consume significant disk I/O and memory. Use frame patches during training and full frames only during evaluation, or use a RAM-backed tmpfs for temporary files.
- **Train-test mismatch in codec version:** If you train with VVenC 1.8 but deploy with VVenC 2.0, the surrogate gradients learned from 1.8's behavior may not transfer. Always match the codec version between training and deployment.

## Limitations

- **Training speed:** Invoking a real codec via subprocess for every forward pass is orders of magnitude slower than a neural proxy. Expect training to take 10-50x longer than proxy-based approaches. GPU utilization will be low during codec calls.
- **Single-frame limitation:** The method as described works best on intra-frame coding. Extending to inter-frame (temporal) coding requires handling GOP structures and temporal dependencies in the surrogate gradient, which adds significant complexity.
- **Codec-specific models:** A downsampler trained with surrogate gradients for x265 will not necessarily perform well with AV1 or VVC. Each target codec requires its own training run.
- **Surrogate gradient approximation quality:** The surrogate is still an approximation of the true codec Jacobian. For highly nonlinear codec behaviors (e.g., mode decisions, in-loop filtering), the local linear assumption may break down.
- **Not suitable for end-to-end rate optimization:** Since the bitrate `R` is also non-differentiable and treated as a constant per QP, the method optimizes distortion at fixed rate points rather than jointly optimizing rate and distortion in a fully differentiable manner.

## Reference

**Paper:** Pesnel, E., Le Tanou, J., Ropert, M., Maugey, T., & Roumy, A. (2026). *SCALED: Surrogate-gradient for Codec-Aware Learning of Downsampling in ABR Streaming*. PCS 2025. [arXiv:2602.00198](https://arxiv.org/abs/2602.00198v1)

**What to look for:** Section on surrogate gradient construction from compression residuals, the custom autograd function implementation, BD-BR evaluation methodology across the R-D convex hull, and the comparison between proxy-codec and surrogate-gradient training showing the 5.19% BD-BR improvement.