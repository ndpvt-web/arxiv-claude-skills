---
name: "opportunities-aiml-rubin-lsst"
description: "Build trustworthy ML pipelines for large-scale scientific data analysis with calibrated uncertainties, simulation-based inference, and physics-informed constraints. Use when: 'build a photometric redshift pipeline', 'add uncertainty quantification to my ML model', 'create a simulation-based inference workflow', 'integrate ML into a scientific analysis pipeline', 'validate ML predictions against simulations', 'set up active learning for survey data'."
---

# Trustworthy Scientific ML Pipelines with Calibrated Inference

This skill enables Claude to design and build ML pipelines for large-scale scientific data analysis that meet the rigor standards outlined in the DESC AI/ML white paper (arXiv:2601.14235). The core principle: scientific ML systems must propagate calibrated uncertainties end-to-end, remain robust under distribution shift between training simulations and real observations, and integrate reproducibly into multi-stage analysis pipelines. This applies broadly to any domain where ML predictions feed into downstream statistical inference — astronomy, climate science, particle physics, genomics, or materials science.

## When to Use

- When the user needs to build an ML pipeline that produces calibrated probability distributions, not just point predictions
- When training data comes from simulations but inference runs on real observations (sim-to-real transfer)
- When ML predictions feed into downstream Bayesian or frequentist statistical analyses requiring proper uncertainty propagation
- When the user asks to implement simulation-based inference (SBI) using neural density estimators
- When building photometric redshift, spectral classification, or other astronomical ML pipelines
- When the user wants to add physics-informed constraints (symmetries, conservation laws, known functional forms) to neural network architectures
- When designing validation suites to detect model misspecification or covariate shift in production ML
- When the user asks to set up active learning loops for expensive data acquisition (telescope time, lab experiments, simulations)

## Key Technique: Trustworthy ML for Precision Science

The DESC white paper identifies a recurring pattern across cosmological probes: ML models trained on simulated data must produce predictions with **calibrated uncertainties** that remain valid when applied to real observations. This is not standard ML — a model with 95% accuracy but miscalibrated confidence intervals can bias downstream cosmological parameter constraints. The paper establishes three pillars for trustworthy scientific ML:

**Pillar 1 — Simulation-Based Inference (SBI):** When the likelihood function is intractable but a simulator exists, SBI methods train neural density estimators (normalizing flows, mixture density networks) to approximate the posterior, likelihood, or likelihood ratio directly from simulated data. The three main variants — Neural Posterior Estimation (NPE), Neural Likelihood Estimation (NLE), and Neural Ratio Estimation (NRE) — each have different trade-offs in amortization, sample efficiency, and composability. NPE gives the posterior directly but must be retrained for new data; NLE learns the likelihood and composes naturally with priors via MCMC; NRE learns density ratios and is most flexible for model comparison.

**Pillar 2 — Calibrated Uncertainty Quantification:** Point predictions are insufficient. The paper emphasizes posterior predictive distributions that are **calibrated** (stated 90% intervals contain the truth 90% of the time) and **sharp** (intervals are as tight as possible). Methods include: deep ensembles for epistemic uncertainty, Monte Carlo dropout, conformal prediction for distribution-free coverage guarantees, and post-hoc recalibration (Platt scaling, isotonic regression, temperature scaling). For photometric redshifts specifically, the standard is to produce full p(z) distributions that pass PIT (Probability Integral Transform) histogram tests.

**Pillar 3 — Physics-Informed Architecture and Loss Design:** Encoding known physics into ML models reduces the hypothesis space, improves generalization, and makes outputs physically interpretable. This includes equivariant neural networks (respecting rotational/translational symmetries), physics-based loss terms (e.g., penalizing violations of conservation laws), hybrid architectures where neural networks learn residuals on top of physical models, and differentiable simulators that enable gradient-based inference through the full forward model.

## Step-by-Step Workflow

1. **Characterize the inference problem.** Determine whether you have: (a) an explicit likelihood — use standard Bayesian methods; (b) a simulator but no tractable likelihood — use SBI; (c) neither — use calibrated discriminative models with uncertainty. Identify what downstream analysis consumes the ML output (e.g., hierarchical Bayesian model, summary statistic compression, cosmological parameter estimation).

2. **Design the simulation-to-observation bridge.** Define the forward model from parameters to observables. Identify the domain gap: what systematic effects exist in real data that simulations may not capture (noise models, selection effects, instrument artifacts)? Plan for domain adaptation or transfer learning to close this gap.

3. **Select and implement the neural density estimator.** For SBI: choose NPE (amortized posterior, fast at inference), NLE (composable with priors, requires MCMC), or NRE (flexible, good for model comparison). Use normalizing flows (e.g., masked autoregressive flows, neural spline flows) as the density estimator backbone. Libraries: `sbi` (Python), `lampe`, `nflows`, or implement with `torch.distributions` and custom flow layers.

4. **Encode physics constraints into the architecture.** Identify symmetries (rotational equivariance for galaxy images, translation invariance for spectra). Use equivariant layers (`e2cnn`, `escnn`) or augmentation-based approaches. Add physics-based regularization to the loss: penalize unphysical predictions (negative fluxes, superluminal velocities). For hybrid models, let the neural network learn corrections to a known physical model rather than learning from scratch.

5. **Implement calibration and uncertainty quantification.** Train an ensemble of 5-10 models with different initializations for epistemic uncertainty. Apply post-hoc calibration on a held-out calibration set using temperature scaling or isotonic regression. For coverage guarantees, implement conformal prediction: compute nonconformity scores on calibration data, then construct prediction sets at the desired confidence level. Validate calibration with PIT histograms, reliability diagrams, and coverage-vs-width plots.

6. **Build the validation suite.** Create a battery of tests that run automatically: (a) calibration metrics (ECE, MCE, PIT uniformity via KS test); (b) distribution shift detection (compare feature distributions of training vs. inference data using MMD or classifier two-sample tests); (c) performance on known-answer simulations where ground truth is available; (d) sensitivity analysis — perturb inputs and check output stability. Encode pass/fail thresholds that gate deployment.

7. **Integrate into the analysis pipeline with uncertainty propagation.** Wrap the ML model as a pipeline stage with standardized I/O: input data format, output posterior samples or parameterized distributions (not just point estimates). Propagate uncertainties to downstream stages — if the next stage is MCMC, provide the learned likelihood or posterior as a callable; if it's a summary statistic, marginalize over the ML uncertainty.

8. **Implement active learning for data acquisition.** If labeling/simulation is expensive, use acquisition functions to select the most informative next data points: (a) maximum entropy (explore uncertain regions); (b) expected information gain (BALD — Bayesian Active Learning by Disagreement); (c) expected posterior contraction for parameter estimation. Run acquisition in batches to amortize overhead.

9. **Set up monitoring and drift detection for production.** Log predictions and their uncertainties. Monitor calibration metrics on incoming data batches. Implement automated alerts when: calibration degrades beyond threshold, input feature distributions shift (detected via running MMD or KS tests), or prediction uncertainty systematically increases. Version all models, training data, and pipeline configurations.

10. **Document assumptions and failure modes.** Record: what simulations the model was trained on, what domain gap mitigation was applied, calibration dataset and results, known failure modes (e.g., out-of-distribution inputs), and the valid operating range of the model. This is non-optional for scientific reproducibility.

## Concrete Examples

**Example 1: Photometric Redshift Pipeline with Calibrated p(z)**

User: "Build a photometric redshift estimation pipeline that outputs calibrated probability distributions for each galaxy."

Approach:
1. Load photometric catalog (ugrizy bands + errors) and spectroscopic training set
2. Train a mixture density network (MDN) to predict p(z|photometry) as a Gaussian mixture
3. Calibrate using conformal prediction on a held-out spec-z sample
4. Validate with PIT histogram and CRPS metrics

Output:
```python
import torch
import torch.nn as nn
from torch.distributions import MixtureSameFamily, Categorical, Normal

class PhotoZMDN(nn.Module):
    """Mixture Density Network for photometric redshift estimation."""
    def __init__(self, n_bands=6, n_components=10, hidden=256):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(n_bands * 2, hidden),  # bands + errors
            nn.SiLU(),
            nn.Linear(hidden, hidden),
            nn.SiLU(),
            nn.Linear(hidden, hidden),
            nn.SiLU(),
        )
        self.weights_head = nn.Linear(hidden, n_components)
        self.means_head = nn.Linear(hidden, n_components)
        self.logvars_head = nn.Linear(hidden, n_components)

    def forward(self, x):
        h = self.net(x)
        weights = Categorical(logits=self.weights_head(h))
        means = self.means_head(h)
        stds = torch.exp(0.5 * self.logvars_head(h)).clamp(min=1e-4)
        components = Normal(means, stds)
        return MixtureSameFamily(weights, components)

    def loss(self, x, z_true):
        dist = self.forward(x)
        return -dist.log_prob(z_true).mean()


def calibrate_with_pit(model, calib_loader):
    """Check calibration via Probability Integral Transform."""
    pit_values = []
    for photometry, z_spec in calib_loader:
        dist = model(photometry)
        pit = dist.cdf(z_spec)
        pit_values.append(pit.detach())
    pit_values = torch.cat(pit_values)
    # Uniform PIT = well-calibrated; test with KS statistic
    from scipy.stats import kstest
    stat, pvalue = kstest(pit_values.numpy(), 'uniform')
    return {"ks_statistic": stat, "p_value": pvalue, "pit_values": pit_values}
```

**Example 2: Simulation-Based Inference for Cosmological Parameters**

User: "I have a cosmological simulator that generates summary statistics given parameters (Omega_m, sigma_8). Set up neural posterior estimation to infer these parameters from observed data."

Approach:
1. Generate training simulations: sample prior, run simulator, collect (params, summaries) pairs
2. Train a normalizing flow to approximate p(params | summaries) via NPE
3. Validate with simulation-based calibration (SBC) checks
4. Apply to observed summary statistics

Output:
```python
import torch
from sbi import utils as sbi_utils
from sbi.inference import SNPE
from sbi.analysis import run_sbc, check_sbc

# 1. Define prior over cosmological parameters
prior = sbi_utils.BoxUniform(
    low=torch.tensor([0.1, 0.5]),   # [Omega_m_min, sigma8_min]
    high=torch.tensor([0.5, 1.2]),  # [Omega_m_max, sigma8_max]
)

# 2. Simulate training data (replace with your simulator)
def simulator(params):
    """Run cosmological forward model. Returns summary statistics."""
    omega_m, sigma8 = params
    # ... call your simulator here ...
    return summary_statistics  # e.g., power spectrum bandpowers

# 3. Train NPE with neural spline flow
inference = SNPE(prior=prior, density_estimator="nsf")
theta, x = sbi_utils.simulate_for_sbi(simulator, prior, num_simulations=50_000)
inference.append_simulations(theta, x)
posterior = inference.train(training_batch_size=256)

# 4. Simulation-Based Calibration check
num_sbc_runs = 1000
thetas_sbc, xs_sbc = sbi_utils.simulate_for_sbi(simulator, prior, num_sbc_runs)
ranks, dap_samples = run_sbc(
    thetas_sbc, xs_sbc, posterior, num_posterior_samples=1000
)
check_stats = check_sbc(ranks, thetas_sbc, dap_samples, num_posterior_samples=1000)
# Uniform rank histograms = well-calibrated posterior

# 5. Infer from real observation
observed_summaries = torch.tensor([...])  # your measured summary statistics
samples = posterior.sample((10_000,), x=observed_summaries)
# samples shape: (10000, 2) — posterior samples of [Omega_m, sigma_8]
```

**Example 3: Physics-Informed Shear Estimation with Equivariant Networks**

User: "Build a galaxy shear estimator that respects rotational symmetry, so rotating the input image rotates the estimated shear accordingly."

Approach:
1. Use SO(2)-equivariant convolutional layers so the network's shear output transforms correctly under rotation
2. Add a physics loss term penalizing unphysical shear magnitudes (|g| > 1)
3. Output a distribution over shear, not a point estimate

Output:
```python
from escnn import nn as enn, gspaces

class EquivariantShearEstimator(torch.nn.Module):
    """SO(2)-equivariant CNN for weak lensing shear estimation."""
    def __init__(self):
        super().__init__()
        self.gspace = gspaces.rot2dOnR2(N=-1)  # continuous SO(2)
        self.in_type = enn.FieldType(self.gspace, [self.gspace.trivial_repr])

        # Equivariant backbone
        feat1 = enn.FieldType(self.gspace, 8 * [self.gspace.regular_repr])
        feat2 = enn.FieldType(self.gspace, 16 * [self.gspace.regular_repr])
        # Output: order-2 representation (spin-2 field = shear)
        self.out_type = enn.FieldType(self.gspace, [self.gspace.irrep(2)])

        self.backbone = enn.SequentialModule(
            enn.R2Conv(self.in_type, feat1, kernel_size=5, padding=2),
            enn.ReLU(feat1),
            enn.PointwiseAvgPool(feat1, 2),
            enn.R2Conv(feat1, feat2, kernel_size=5, padding=2),
            enn.ReLU(feat2),
            enn.PointwiseAvgPool(feat2, 2),
            enn.R2Conv(feat2, self.out_type, kernel_size=5, padding=2),
        )
        # Global pool to get single shear per galaxy
        self.pool = enn.GroupPooling(self.out_type)

    def forward(self, image):
        x = self.in_type(image)
        features = self.backbone(x)
        shear = self.pool(features).tensor  # (batch, 2): [g1, g2]
        return shear

    def physics_loss(self, shear_pred, shear_true):
        mse = ((shear_pred - shear_true) ** 2).mean()
        # Penalize unphysical |g| > 1
        magnitude = torch.sqrt((shear_pred ** 2).sum(dim=-1))
        penalty = torch.relu(magnitude - 1.0).mean()
        return mse + 10.0 * penalty
```

## Best Practices

- **Do:** Always output full posterior distributions or prediction sets, never just point estimates. Downstream science requires proper uncertainty propagation.
- **Do:** Validate calibration on held-out data using PIT histograms, reliability diagrams, and simulation-based calibration (SBC) rank checks. Passing training loss is insufficient.
- **Do:** Encode known physics (symmetries, conservation laws, valid ranges) directly into the architecture or loss rather than hoping the network learns them from data.
- **Do:** Version everything — model weights, training data, simulator version, hyperparameters, calibration datasets. Scientific reproducibility demands it.
- **Avoid:** Training on simulations and deploying on real data without explicit domain gap mitigation. Always test for distribution shift using two-sample tests (MMD, classifier two-sample test) on input features.
- **Avoid:** Using a single model without ensembling or other epistemic uncertainty estimation. A single network's softmax probabilities are not calibrated uncertainties.

## Error Handling

| Problem | Detection | Mitigation |
|---------|-----------|------------|
| PIT histogram non-uniform (miscalibrated p(z)) | KS test p-value < 0.01 | Apply post-hoc recalibration (isotonic regression or temperature scaling) on calibration set |
| SBC rank histogram non-uniform (biased posterior) | Chi-squared test on rank bins | Retrain with more simulations, check simulator fidelity, use multi-round SBI (SNPE-C) |
| Covariate shift detected at inference time | MMD or classifier two-sample test flags shift | Apply domain adaptation, retrain with augmented simulations, or fall back to wider priors |
| Normalizing flow training unstable (NaN losses) | Loss divergence during training | Reduce learning rate, use gradient clipping, switch to neural spline flows (more numerically stable than affine coupling) |
| Overconfident predictions on OOD inputs | Uncertainty is low but predictions are wrong | Add OOD detection layer (Mahalanobis distance, deep kernel density), flag and reject OOD inputs |
| Physics constraint violations in outputs | Post-prediction validation check | Increase physics penalty weight in loss, project outputs to valid manifold post-hoc |

## Limitations

- **SBI requires a fast, accurate simulator.** If your forward model takes hours per evaluation, you need emulators or surrogate models first, which introduces its own approximation errors.
- **Calibration is conditional.** A model calibrated on one population may not be calibrated on subsets. Photometric redshifts calibrated globally can be miscalibrated for rare galaxy types.
- **Physics-informed constraints require known physics.** If the relevant physics is uncertain or the symmetry is approximate, hard-coding it can introduce bias. Use soft constraints (loss penalties) rather than hard constraints (architectural) when physics is approximate.
- **Conformal prediction guarantees are marginal, not conditional.** Distribution-free coverage holds on average across the test set, not necessarily for each individual input. Conditional coverage requires additional assumptions.
- **Foundation models for scientific data are nascent.** The paper discusses the potential of foundation models and LLM-driven agentic systems but notes deployment must be "coupled with rigorous evaluation and governance." Do not assume astronomical foundation models are production-ready.
- **This approach requires substantial simulation budgets.** SBI typically needs 10K-100K+ simulations for training. Active learning can reduce this but adds pipeline complexity.

## Reference

**Paper:** "Opportunities in AI/ML for the Rubin LSST Dark Energy Science Collaboration" (arXiv:2601.14235v1, 84 pages, 2026)
**What to look for:** Sections on cross-cutting ML challenges (uncertainty quantification, covariate shift, validation), per-probe ML surveys (photo-z, weak lensing, SN classification, LSS, clusters, strong lensing, theory/simulations), and the infrastructure/governance recommendations. The paper's key value is mapping which ML methods recur across multiple science cases, identifying them as high-priority shared investments.