---
name: "1100-high-efficiency-visual-adapter-complex"
description: "Implement CoLin (Complex Linear Projection) adapters for parameter-efficient fine-tuning of vision foundation models. Adds ~1% trainable parameters to frozen backbones while outperforming full fine-tuning. Use when: 'add CoLin adapter to my vision model', 'parameter-efficient fine-tuning for detection', 'adapt a frozen Swin Transformer', 'low-rank adapter with orthogonal loss', 'efficient visual adapter for segmentation', 'fine-tune vision model with 1% parameters'."
---

# CoLin: High-Efficiency Visual Adapter with Complex Linear Projection

This skill enables Claude to implement CoLin adapters -- a parameter-efficient fine-tuning method that decomposes adapter weight matrices into a multi-branch low-rank structure `W = sum(P_i^T K_i Q_i)` with SVD-based initialization and orthogonal regularization. CoLin adds only ~1% trainable parameters to a frozen vision backbone (Swin Transformer, ViT, etc.) and consistently outperforms both full fine-tuning and classical delta-tuning methods like LoRA and AdaptFormer on detection, segmentation, classification, and rotated object detection tasks.

## When to Use

- When the user wants to fine-tune a vision foundation model (Swin, ViT, DINOv2) on a downstream task without updating all parameters
- When adapting a pretrained detection/segmentation backbone (e.g., Swin-B/L for COCO, VOC, ADE20K) with minimal GPU memory
- When implementing a low-rank adapter that needs to outperform LoRA or AdaptFormer on vision tasks
- When the user mentions parameter-efficient transfer learning (PETL/PEFT) for computer vision specifically
- When deploying vision models to resource-constrained environments where storing full fine-tuned weights per task is impractical
- When working with MMDetection, MMSegmentation, or similar frameworks and needing adapter integration into transformer blocks

## Key Technique

**Architecture.** CoLin decomposes the adapter weight matrix `W` into a multi-branch low-rank form: `W = sum_{i=1}^{alpha} P_i^T K_i Q_i`, where `P in R^{beta x m}`, `K in R^{beta x beta}` (diagonal scaling), and `Q in R^{beta x n}`. The rank `beta` (typically 28) and branch count `alpha` (typically 4) are hyperparameters. A key efficiency trick is **sharing**: `P` and `Q` matrices are shared across all branches, with only the diagonal `K_i` (called `g_i` in code) varying per branch. This keeps parameter count at ~1% of the backbone. Two CoLin modules are inserted per transformer block -- one after the attention residual, one after the FFN residual -- with a gated depthwise convolution between the down and up projections to capture spatial information.

**Convergence fix.** The core theoretical contribution is proving that composite low-rank matrices `W = PQ` suffer from gradient direction entanglement: the effective update becomes `Delta_W ~ -eta * grad_W * (QQ^T + P^T P)` instead of the ideal `-eta * grad_W`. When `P^T P` and `QQ^T` deviate from identity, optimization diverges from the true gradient direction. CoLin fixes this with an orthogonal regularization loss: `L_orth = ||P^T P - I||_F^2 + ||QQ^T - I||_F^2`, added to the task loss as `L = L_task + lambda * sum(L_orth)` across all adapter layers. This ensures the composite update direction stays aligned with the ideal gradient.

**Initialization.** Matrices are initialized via SVD decomposition of Kaiming-uniform random matrices. For each branch, a random `W_0` is decomposed as `U, S, V = SVD(W_0)`, then `P` is set from `U[:, :beta]`, `g_i` from `S[:beta]`, and `Q` from `V^T[:beta, :]`. The shared `P` and `Q` are averaged across branches. This ensures near-orthogonality from the start, making the orthogonal loss effective rather than fighting random initialization.

## Step-by-Step Workflow

1. **Freeze the backbone.** Load pretrained weights (e.g., Swin-B ImageNet-22K) and set `requires_grad=False` for all backbone parameters. Only adapter parameters will be trainable.

2. **Define the CLP (Complex Linear Projection) layer.** Create a `CLP_Multi` module with shared `A` (P matrix, shape `[in_features, kernel_dim]`), shared `B` (Q matrix, shape `[kernel_dim, out_features]`), and per-branch diagonal scales `g_i` (shape `[kernel_dim, 1]`). The forward pass computes `weight = sum_i (A @ diag(g_i) @ B).T` then applies `F.linear(x, weight)`.

3. **Build the CoLin adapter module.** Compose two `CLP_Multi` layers (down-projection to `INNER_DIM`, up-projection back) with a gated depthwise `Conv2d` in between. Use `INNER_DIM = in_dim // 4`, `KERNEL_DIM = 28`, `NUM_BRANCH = 4`. Add a learnable `inner_scale` parameter for the sigmoid gate on the convolution path.

4. **Initialize with SVD.** In `reset_parameters()`, for each branch: generate a Kaiming-uniform random matrix, compute its SVD, extract `U[:, :kernel_dim]`, `S[:kernel_dim]`, `V^T[:kernel_dim, :]`. Average the U and V components across branches for the shared A and B. Assign S values to each branch's `g_i`.

5. **Insert adapters into transformer blocks.** Add `self.colin1 = CoLin(embed_dims)` and `self.colin2 = CoLin(embed_dims)` to each transformer block. Call `colin1` after the attention + residual connection, and `colin2` after the FFN + residual connection. Both use additive residual: `output = x + colin(x)`.

6. **Implement the orthogonal loss.** Traverse all `CLP_Multi` modules, compute `||A^T A - I||_F^2 + ||B B^T - I||_F^2` for each, and sum them. Add to the task loss with weight `lambda` (start with `lambda = 0.1`, tune per task).

7. **Integrate orthogonal loss into training loop.** In the custom runner or training hook, call a `get_orthogonal_loss()` method on the backbone after each forward pass. Add `lambda * orth_loss` to the total loss before `backward()`.

8. **Configure training hyperparameters.** Use the same optimizer and schedule as full fine-tuning for the given task (e.g., AdamW, 3x schedule for COCO detection). Set `norm_cfg = SyncBN` for multi-GPU. Only adapter params + any unfrozen norm layers go into the optimizer param groups.

9. **Enable inference-time weight fusion.** At eval time, precompute `rep_weight = sum_i (A @ diag(g_i) @ B).T` once per layer and use it as a static linear weight. This eliminates the multi-branch overhead at inference, making forward pass cost identical to a standard linear layer.

10. **Validate parameter count.** After model construction, count trainable parameters and verify they are ~1-2% of the backbone. Print a summary: `trainable = sum(p.numel() for p in model.parameters() if p.requires_grad)`.

## Concrete Examples

**Example 1: Adding CoLin adapter to Swin-B for COCO object detection**

User: "I have a pretrained Swin-B backbone and want to fine-tune it on COCO with Cascade Mask R-CNN using minimal parameters."

Approach:
1. Load Swin-B pretrained weights and freeze all backbone layers
2. Implement the CoLin adapter modules as described above
3. Modify each `SwinBlock` to include two CoLin adapters
4. Add orthogonal loss collection to the training pipeline
5. Train with MMDetection's 3x schedule

Key code for the adapter module:

```python
INNER_DIM = 48  # embed_dims // factor, with factor=4 for dim=192
KERNEL_DIM = 28
NUM_BRANCH = 4

class CLP_Multi(nn.Module):
    def __init__(self, in_features, out_features, num_branch, kernel_dim, up_proj=None):
        super().__init__()
        self.in_features = in_features
        self.out_features = out_features
        self.num_branch = num_branch
        self.kernel_dim = kernel_dim
        self.sharing = up_proj is not None

        self.param_dict = nn.ParameterDict()
        if self.sharing and up_proj is not None:
            # Branch sharing: reuse A, B from the up-projection
            self.param_dict["A"] = up_proj.param_dict["A"]
            self.param_dict["B"] = up_proj.param_dict["B"]
        else:
            self.param_dict["A"] = nn.Parameter(torch.empty(in_features, kernel_dim))
            self.param_dict["B"] = nn.Parameter(torch.empty(kernel_dim, out_features))

        for i in range(num_branch):
            self.param_dict[f"g{i}"] = nn.Parameter(torch.empty(kernel_dim, 1))

        self.reset_parameters()

    def reset_parameters(self):
        As, Bs = 0, 0
        for i in range(self.num_branch):
            W = torch.empty(self.in_features, self.out_features)
            nn.init.kaiming_uniform_(W, a=math.sqrt(5))
            U, S, Vt = torch.linalg.svd(W, full_matrices=False)
            As = As + U[:, :self.kernel_dim]
            self.param_dict[f"g{i}"].data = S[:self.kernel_dim].unsqueeze(1)
            Bs = Bs + Vt[:self.kernel_dim, :]
        if not self.sharing:
            self.param_dict["A"].data = As / self.num_branch
            self.param_dict["B"].data = Bs / self.num_branch

    def forward(self, x):
        A = self.param_dict["A"]
        B = self.param_dict["B"]
        weight = sum(
            (A @ torch.diag(self.param_dict[f"g{i}"].squeeze()) @ B).T
            for i in range(self.num_branch)
        )
        return F.linear(x, weight)

    def orthogonal_loss(self):
        A = self.param_dict["A"]
        B = self.param_dict["B"]
        I_a = torch.eye(A.size(1), device=A.device)
        I_b = torch.eye(B.size(0), device=B.device)
        return torch.norm(A.T @ A - I_a, p="fro") ** 2 + torch.norm(B @ B.T - I_b, p="fro") ** 2


class CoLin(nn.Module):
    def __init__(self, in_dim, factor=4):
        super().__init__()
        inner_dim = in_dim // factor
        self.project1 = CLP_Multi(in_dim, inner_dim, NUM_BRANCH, KERNEL_DIM)
        self.project2 = CLP_Multi(inner_dim, in_dim, NUM_BRANCH, KERNEL_DIM,
                                   up_proj=self.project1)
        self.inner_op = nn.Conv2d(inner_dim, inner_dim, 3, padding=1, groups=inner_dim)
        self.inner_scale = nn.Parameter(torch.zeros(1))

    def forward(self, x, hw_shape):
        b, n, c = x.shape
        h, w = hw_shape
        out = self.project1(x)
        out_2d = out.view(b, h, w, -1).permute(0, 3, 1, 2)
        scale = torch.sigmoid(self.inner_scale)
        out_2d = scale * self.inner_op(out_2d) + (1.0 - scale) * out_2d
        out = out_2d.permute(0, 2, 3, 1).view(b, n, -1)
        out = F.gelu(out)
        out = self.project2(out)
        return x + out  # residual
```

Expected result: ~1.71M trainable params (1.97% of Swin-B), achieving 52.9% box AP and 45.5% mask AP on COCO -- surpassing full fine-tuning (52.4% / 45.1%).

---

**Example 2: Collecting and applying orthogonal loss in a training loop**

User: "How do I add the orthogonal regularization loss to my existing PyTorch training loop?"

Approach:
1. Write a helper that traverses the model for all `CLP_Multi` modules
2. Sum their orthogonal losses
3. Add the weighted sum to the task loss

```python
def collect_orthogonal_loss(model, lambda_orth=0.1):
    orth_loss = 0.0
    for module in model.modules():
        if isinstance(module, CLP_Multi):
            orth_loss += module.orthogonal_loss()
    return lambda_orth * orth_loss

# In training loop:
for batch in dataloader:
    outputs = model(batch["images"])
    task_loss = criterion(outputs, batch["targets"])
    orth_loss = collect_orthogonal_loss(model, lambda_orth=0.1)
    total_loss = task_loss + orth_loss
    total_loss.backward()
    optimizer.step()
    optimizer.zero_grad()
```

---

**Example 3: Adapting CoLin for a ViT backbone (non-Swin)**

User: "I want to use CoLin with a plain ViT-B/16 for image classification on CIFAR-100."

Approach:
1. Since ViT has no spatial window structure, drop the depthwise conv path (or use 1D conv)
2. Insert CoLin after each ViT block's attention and MLP residual connections
3. Adjust INNER_DIM for ViT-B (dim=768, so INNER_DIM=192 with factor=4)

```python
class CoLinForViT(nn.Module):
    """CoLin without spatial conv, suitable for ViT with 1D token sequences."""
    def __init__(self, in_dim, factor=4):
        super().__init__()
        inner_dim = in_dim // factor
        self.project1 = CLP_Multi(in_dim, inner_dim, NUM_BRANCH, KERNEL_DIM)
        self.project2 = CLP_Multi(inner_dim, in_dim, NUM_BRANCH, KERNEL_DIM,
                                   up_proj=self.project1)

    def forward(self, x):
        out = self.project1(x)
        out = F.gelu(out)
        out = self.project2(out)
        return x + out

# Monkey-patch into ViT blocks:
for block in vit_model.blocks:
    block.colin1 = CoLinForViT(768)
    block.colin2 = CoLinForViT(768)
    original_forward = block.forward

    def patched_forward(self, x, colin1=block.colin1, colin2=block.colin2, orig=original_forward):
        # Attention + residual
        x = orig.__wrapped_attn__(x)  # your attention call
        x = colin1(x)
        # FFN + residual
        x = orig.__wrapped_ffn__(x)
        x = colin2(x)
        return x
```

Expected result: ~1.2% additional parameters, competitive with or exceeding AdaptFormer on classification benchmarks.

## Best Practices

- **Do:** Always initialize P and Q via SVD decomposition of random matrices. Random or zero initialization causes the orthogonal loss to fight uphill from the start, delaying convergence significantly.
- **Do:** Share P and Q matrices between the down-projection and up-projection within each CoLin module (kernel sharing). This halves the adapter parameter count with negligible accuracy loss.
- **Do:** Fuse multi-branch weights into a single `rep_weight` matrix at inference time. This eliminates the branch loop overhead and makes the adapter equivalent to a standard `nn.Linear`.
- **Do:** Keep `KERNEL_DIM` (rank beta) around 28 and `NUM_BRANCH` at 4. The paper's ablations show diminishing returns beyond these values.
- **Avoid:** Applying CoLin to already-unfrozen layers. The entire point is adapting a frozen backbone -- if layers are unfrozen, the orthogonal constraint on adapter matrices may conflict with backbone gradient updates.
- **Avoid:** Setting `lambda_orth` too high (>1.0). This over-constrains P and Q to stay orthogonal at the expense of task performance. Start at 0.1 and tune downward if validation metrics plateau early.

## Error Handling

- **SVD fails on GPU with large matrices:** Use `torch.linalg.svd` with `full_matrices=False` to compute only the top-k singular vectors. If memory is tight, run initialization on CPU and transfer to GPU afterward.
- **Orthogonal loss explodes early in training:** This usually means initialization was not SVD-based. Check that `reset_parameters()` is called and that shared parameters are properly linked (not duplicated).
- **No improvement over LoRA:** Verify that the gated depthwise convolution is active (check `inner_scale` is not stuck at 0). Also confirm that the orthogonal loss is actually being added to the total loss -- a common bug is collecting it but forgetting to add it before `backward()`.
- **Parameter count higher than expected:** Check for accidental unfreezing of backbone params. Use `sum(p.numel() for p in model.parameters() if p.requires_grad)` to audit. Also verify that shared A/B matrices are truly shared (same `id()`) and not independent copies.
- **NaN in orthogonal loss:** Can occur if `kernel_dim` exceeds the matrix rank. Ensure `kernel_dim < min(in_features, out_features)`.

## Limitations

- CoLin adapters add inference latency since they are sequential modules (unlike LoRA which merges into existing weights). The reparameterization at eval time mitigates this but the depthwise conv path remains.
- The method is validated primarily on Swin Transformer backbones with MMDetection. Adapting to other architectures (ConvNeXt, ViT-based MAE, etc.) requires manual integration into each block type.
- Semantic segmentation and classification code is marked "TBD" in the official repo -- only detection configs are fully released as of the paper's publication.
- The orthogonal loss adds a hyperparameter (`lambda`) that requires per-task tuning, unlike LoRA which is closer to plug-and-play.
- Multi-branch weight fusion at inference assumes static input dimensions. Dynamic input shapes (common in detection) work fine, but the fused weight must be recomputed if adapter structure changes at runtime.

## Reference

**Paper:** [1%>100%: High-Efficiency Visual Adapter with Complex Linear Projection Optimization](https://arxiv.org/abs/2602.10513v1) -- Yin et al., 2026. Focus on Section 3 (method) for the W = P^T K Q decomposition and the gradient entanglement proof, Section 4 for the orthogonal loss derivation, and Tables 1-4 for benchmark results across detection/segmentation/classification.

**Code:** [github.com/DongshuoYin/CoLin](https://github.com/DongshuoYin/CoLin) -- `MMDetection/mmdet/models/backbones/swin_colin.py` contains the core adapter implementation.