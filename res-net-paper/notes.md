# ResNet — Deep Residual Learning for Image Recognition

He, Zhang, Ren, Sun (Microsoft Research, 2015). arXiv:1512.03385. Won ImageNet 2015.

## The problem: degradation
- Naively stacking more layers made networks *worse* — higher **training** error (not overfitting, not vanishing gradients, which BN had mostly fixed).
- Puzzle: a deep net should be able to match a shallow one by making extra layers do nothing (identity). But gradient descent couldn't find that solution — learning to "pass input through unchanged" is hard for the optimizer.

## The key idea: residual learning
- Instead of a block learning the full target `H(x)`, have it learn the **residual** `F(x) = H(x) - x`, then add the input back: `output = F(x) + x`.
- Implemented with a **shortcut / skip connection** — a wire that jumps over the block and adds `x` to its output (Fig. 2).
- Why it works:
  - If "do nothing" is optimal, just push `F(x)` toward **zero** — easy, vs. learning an exact identity through nonlinear layers.
  - Depth can no longer hurt: extra blocks trivially learn ~zero and pass info through.
  - Free: shortcut is just addition — no extra params or compute; trains with normal backprop.
- Analogy: each layer proposes small *edits* to what it received rather than rewriting from scratch.

## Architecture (Fig. 3)
- VGG-inspired plain net (3×3 filters) + skip connections every ~2 layers.
- Matching dims → identity shortcut (0 params). Changing dims → zero-pad or 1×1 conv to align shapes for the add.
- Deeper variants (50/101/152) use **bottleneck** blocks (1×1 → 3×3 → 1×1).

## Results
- Plain 34-layer worse than plain 18-layer (degradation confirmed).
- ResNet 34-layer *beat* 18-layer — depth finally helped.
- 152-layer ResNet: deepest on ImageNet at the time, still cheaper than VGG-19. Ensemble: **3.57% top-5 error**.

## Why it matters
- Skip connections are now everywhere: Transformers/LLMs, diffusion models, etc.
- Core lesson: make it easy for layers to do nothing, so added depth never hurts.

## Companion notebooks
- [gradients_explained.ipynb](gradients_explained.ipynb) — visual explainer of vanishing/exploding gradients (why gradients are a product, a real deep net, init fixes, and how skip connections rescue gradient flow).
- [backpropagation_explained.ipynb](backpropagation_explained.ipynb) — beginner walkthrough of backprop (loss as a bowl, gradient = downhill direction, the chain rule, a tiny net trained by hand). Explains where the per-layer "factors" in the gradients notebook come from.

### Running the notebooks
- Project venv + Jupyter kernel already set up: `.venv/` (gitignored), kernel name **`resnet-notes`** (display: "Python (res-net-paper)").
- VS Code/Cursor: open a `.ipynb`, pick kernel **"Python (res-net-paper)"**, run with Shift+Enter.
- Browser: `./.venv/bin/jupyter lab` then select the `resnet-notes` kernel.
- Recreate the venv if `.venv/` is gone: `python3 -m venv .venv && ./.venv/bin/pip install numpy matplotlib ipykernel jupyter && ./.venv/bin/python -m ipykernel install --user --name resnet-notes --display-name "Python (res-net-paper)"`
