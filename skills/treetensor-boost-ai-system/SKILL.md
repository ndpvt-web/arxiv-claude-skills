---
name: "treetensor-boost-ai-system"
description: >
  Implement TreeTensor-based nested data handling for AI systems using the DI-treetensor library.
  Replaces manual recursive traversal of hierarchical data (dicts-of-tensors, nested observations,
  multi-agent states) with tree-structured tensor operations that auto-propagate through the hierarchy.
  Use this skill when:
  - "Help me batch nested dictionary observations for RL training"
  - "Stack variable-structure game states into a training batch"
  - "Apply a neural network to nested observation data without manual loops"
  - "Simplify my multi-agent data pipeline that has deeply nested dicts"
  - "Convert my recursive tensor processing code to use TreeTensor"
  - "Handle heterogeneous nested data in PyTorch without boilerplate"
---

# TreeTensor: Nested Data Handling for AI Systems

This skill enables Claude to refactor and build AI data pipelines that handle hierarchical (nested) data using the TreeTensor library (`di-treetensor`). Instead of writing custom recursive functions to traverse dicts-of-tensors, TreeTensor wraps nested structures into constrained tree containers where standard operations (arithmetic, stacking, reshaping, gradient computation) automatically propagate to every leaf tensor. This eliminates boilerplate, reduces bugs, and matches the performance of hand-written code with zero overhead.

## When to Use

- When the user has nested dictionaries of PyTorch tensors (e.g., RL observations with `{'obs': {'scalar': tensor, 'image': tensor}, 'action': tensor}`) and needs to batch, stack, or transform them
- When the user is writing recursive helper functions to apply operations across nested data structures containing tensors
- When building multi-agent systems (like AlphaStar-style architectures) where each agent produces heterogeneous nested state
- When the user needs to collate variable-structure data into training batches without writing custom `collate_fn` logic
- When converting between nested-dict and batched-tensor representations for model input/output
- When the user wants to apply PyTorch operations (reshape, to device, gradient tracking) uniformly across all tensors in a nested structure

## Key Technique

**The Problem.** In complex AI systems -- especially reinforcement learning, game AI, and multi-modal pipelines -- data is naturally hierarchical. A single environment step might produce nested dicts like `{'obs': {'visual': tensor(3,64,64), 'scalar': tensor(12)}, 'reward': tensor(1), 'info': {'health': tensor(1)}}`. Standard PyTorch tensors require fixed shapes, so developers write recursive traversal functions to stack, move-to-device, or transform these structures. This code is fragile, hard to maintain, and must be rewritten for every new data schema.

**The TreeTensor Solution.** TreeTensor models nested data as a constrained tree where internal nodes are dict-like branches and leaf nodes are actual tensors (or scalars). The library overloads Python's magic methods (`__add__`, `__mul__`, etc.) and wraps PyTorch/NumPy functions so that any operation applied to a tree automatically maps element-wise across all corresponding leaves. The key constraint is structural compatibility: two trees being combined must share the same branching structure (keys at each level must match). This constraint enables safe, automatic operation dispatch without explicit recursion.

**Two Computational Patterns.** The paper identifies (1) *element-wise operations* -- applying the same function to corresponding leaves of aligned trees (e.g., `tree_a + tree_b` adds matching leaf tensors), and (2) *cross-element operations* -- aggregating across multiple trees (e.g., `ttorch.stack([tree1, tree2, ...])` stacks all corresponding leaves along a new batch dimension). Both patterns are handled transparently by the library with near-zero overhead thanks to the Cython-optimized `FastTreeValue` engine underneath.

## Step-by-Step Workflow

1. **Install the library.** Run `pip install di-treetensor`. For serialization support, use `pip install di-treetensor[potc]`.

2. **Identify nested data structures.** Locate code where dictionaries (or nested dicts) of tensors are created, passed around, or processed. Look for patterns like `data['obs']['image']` or recursive helper functions that walk nested dicts.

3. **Wrap raw dicts into TreeTensors.** Convert nested dictionaries to tree tensors at the data boundary:
   ```python
   import treetensor.torch as ttorch
   tree_data = ttorch.tensor({
       'obs': {'scalar': [1.0, 2.0], 'image': torch.randn(3, 32, 32)},
       'action': [3],
       'reward': [0.5],
   })
   ```

4. **Replace recursive operations with direct tree operations.** Delete custom recursive `stack()`, `to_device()`, or `apply_fn()` helpers. Use TreeTensor's built-in operations instead:
   - Stacking: `ttorch.stack([tree1, tree2, tree3], dim=0)`
   - Device transfer: `tree_data = tree_data.cuda()` or `tree_data.to(device)`
   - Arithmetic: `normalized = (tree_data - tree_mean) / tree_std`

5. **Apply PyTorch functions directly to trees.** Most PyTorch operations work on tree tensors out of the box:
   ```python
   result = ttorch.sin(tree_data)          # element-wise sin on all leaves
   reshaped = tree_data.reshape(-1)         # reshape every leaf
   tree_data.requires_grad_(True)           # enable grad tracking on all leaves
   ```

6. **Access individual leaves when needed.** Use attribute access to reach specific tensors for operations that should not propagate:
   ```python
   just_image = tree_data.obs.image         # returns a plain torch.Tensor
   action_logits = model.action_head(tree_data.obs.scalar)  # standard PyTorch call
   ```

7. **Use `ttorch.stack` for batching environment steps.** Collect a list of tree-tensor transitions, then batch them in one call:
   ```python
   batch = ttorch.stack([ttorch.tensor(env.step(a)) for a in actions], dim=0)
   ```

8. **Inspect tree structure for debugging.** Print a tree tensor to see its full hierarchy with shapes:
   ```python
   print(tree_data)
   # <Tensor 0x...>
   # +-- obs
   # |   +-- scalar: tensor([1.0, 2.0])
   # |   +-- image: tensor(shape=(3, 32, 32))
   # +-- action: tensor([3])
   # +-- reward: tensor([0.5])
   ```

9. **Handle custom leaf types with `subside` and `rise`.** For advanced cases where you need to convert between a tree-of-tensors and a single batched tensor (or vice versa), use the `subside`/`rise` utilities documented in the library.

10. **Validate structural compatibility.** When combining two trees, ensure they share the same key structure. Mismatched keys will raise errors. Add assertions or use `tree.keys()` comparisons at data boundaries.

## Concrete Examples

**Example 1: Batching RL environment transitions**

User: "I have a list of environment transitions, each a nested dict with obs/action/reward/done. I need to stack them into a batch for training. Right now I have a recursive stack function that keeps breaking when I add new fields."

Approach:
1. Wrap each transition dict with `ttorch.tensor()`
2. Call `ttorch.stack()` on the list
3. Delete the old recursive `stack()` helper

```python
# BEFORE -- fragile recursive helper
def recursive_stack(data_list, dim=0):
    elem = data_list[0]
    if isinstance(elem, torch.Tensor):
        return torch.stack(data_list, dim)
    elif isinstance(elem, dict):
        return {k: recursive_stack([d[k] for d in data_list], dim) for k in elem}
    elif isinstance(elem, bool):
        return torch.BoolTensor(data_list)
    else:
        raise TypeError(f"Unsupported: {type(elem)}")

transitions = [env.step(a) for a in actions]
batch = recursive_stack(transitions)

# AFTER -- one-liner with TreeTensor
import treetensor.torch as ttorch

transitions = [ttorch.tensor(env.step(a)) for a in actions]
batch = ttorch.stack(transitions, dim=0)
# batch.obs.image.shape -> (B, 3, 64, 64)
# batch.reward.shape    -> (B, 1)
```

Output: A single tree tensor where every leaf is batched along dim=0. Adding new fields to the environment step requires zero code changes.

---

**Example 2: Moving multi-agent state to GPU**

User: "I have a nested dict of tensors representing game state for multiple agents. I want to move everything to GPU without writing a recursive to() function."

Approach:
1. Wrap the state dict as a tree tensor
2. Call `.cuda()` or `.to(device)` directly

```python
import treetensor.torch as ttorch

state = ttorch.tensor({
    'agents': {
        'positions': torch.randn(8, 3),
        'health': torch.randn(8),
        'inventory': {
            'items': torch.randint(0, 100, (8, 10)),
            'counts': torch.randint(0, 5, (8, 10)),
        },
    },
    'global': {
        'time_step': torch.tensor([42]),
        'score': torch.tensor([1500.0]),
    },
})

# One call moves everything
state_gpu = state.cuda()

# Verify
print(state_gpu.agents.inventory.items.device)  # cuda:0
```

---

**Example 3: Normalizing nested observations**

User: "I need to normalize all observation tensors by their running mean and std. The observations are deeply nested dicts."

Approach:
1. Maintain running mean/std as tree tensors with matching structure
2. Apply arithmetic directly

```python
import treetensor.torch as ttorch

# Assume running_mean and running_std are tree tensors with same structure as obs
obs = ttorch.tensor({
    'visual': torch.randn(4, 3, 64, 64),
    'proprioception': {
        'joint_angles': torch.randn(4, 12),
        'velocities': torch.randn(4, 12),
    },
})

running_mean = ttorch.tensor({
    'visual': torch.zeros(3, 64, 64),
    'proprioception': {
        'joint_angles': torch.zeros(12),
        'velocities': torch.zeros(12),
    },
})

running_std = ttorch.tensor({
    'visual': torch.ones(3, 64, 64),
    'proprioception': {
        'joint_angles': torch.ones(12),
        'velocities': torch.ones(12),
    },
})

# Normalize -- broadcasts and applies to every leaf automatically
normalized_obs = (obs - running_mean) / (running_std + 1e-8)
```

## Best Practices

- **Do:** Wrap data into tree tensors at system boundaries (environment output, data loader) and keep them as trees throughout the pipeline. Converting back and forth defeats the purpose.
- **Do:** Use `ttorch.stack` and `ttorch.cat` instead of writing custom collation logic. These handle arbitrary nesting depths automatically.
- **Do:** Access `.shape` on a tree tensor to get a tree of shapes for quick debugging of the entire hierarchy at once.
- **Do:** Combine TreeTensor with standard PyTorch modules -- extract individual leaf tensors via attribute access (e.g., `batch.obs.image`) before feeding into `nn.Conv2d` layers that expect plain tensors.
- **Avoid:** Mixing tree tensors with incompatible structures in binary operations. Both operands must share the same key hierarchy, or you will get a `KeyError`.
- **Avoid:** Using TreeTensor for flat, homogeneous data that fits naturally into a single tensor. The library adds value only when data is genuinely nested or heterogeneous.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| `KeyError` during tree operation | Two trees have different key structures | Ensure both trees share identical nesting hierarchy; add missing keys or restructure data |
| `RuntimeError: shape mismatch` | Leaf tensors at corresponding positions have incompatible shapes | Check `.shape` on both trees and ensure compatible dimensions for the intended operation |
| `TypeError: not support elem type` | Non-tensor leaf values (strings, None) in the dict | Filter or convert non-tensor leaves before wrapping with `ttorch.tensor()` |
| Slow performance on very deep trees | Excessive nesting depth (>10 levels) | Flatten unnecessary intermediate levels; TreeTensor is optimized for moderate depth |
| `ImportError` for treetensor | Library not installed | `pip install di-treetensor` |

## Limitations

- **Fixed key structure required for binary ops.** Two tree tensors must have exactly matching key hierarchies to be combined. You cannot add a tree with keys `{a, b}` to one with keys `{a, c}` -- there is no automatic key union or intersection.
- **Not a replacement for PyTorch modules.** TreeTensor handles data containers, not model architecture. Neural network layers still operate on plain tensors; you must extract leaves before feeding them into `nn.Module` layers.
- **Limited ecosystem adoption.** While the library wraps PyTorch and NumPy, not all third-party libraries accept tree tensors. You may need to convert to plain dicts at integration boundaries.
- **Dynamic tree structures.** If your nested data changes shape across steps (e.g., variable number of agents adding/removing keys), TreeTensor's structural constraints may require padding or restructuring.
- **Best suited for moderate nesting.** Trees with 2-5 levels of nesting benefit most. Extremely deep hierarchies (10+ levels) or extremely wide ones (1000+ keys at one level) may not see practical gains.

## Reference

**Paper:** [TreeTensor: Boost AI System on Nested Data with Constrained Tree-Like Tensor](https://arxiv.org/abs/2602.08517v1) by Shaoang Zhang and Yazhe Niu (2026). Key sections: Section 3 for the two computational patterns (element-wise and cross-element), Section 4 for the constrained tree formalization, and Section 5 for the AlphaStar benchmark showing real-world application in complex game AI.

**Library:** [github.com/opendilab/DI-treetensor](https://github.com/opendilab/DI-treetensor) -- install with `pip install di-treetensor`.