---
name: "eliciting-least-to-most-reasoning-phishing"
description: "Detect phishing URLs using Least-to-Most iterative decomposition with answer sensitivity scoring. Triggers: 'analyze this URL for phishing', 'is this URL safe', 'check URL for phishing indicators', 'phishing detection', 'classify this URL as phishing or benign', 'scan URL for suspicious patterns'"
---

# Least-to-Most Reasoning for Phishing URL Detection

This skill enables Claude to classify URLs as phishing or benign using a **Least-to-Most prompting framework with answer sensitivity**. Rather than making a single snap judgment, Claude decomposes URL analysis into a series of increasingly specific sub-questions — examining domain structure, path components, TLD patterns, and brand impersonation signals — where each answer informs the next. A running phishing-likelihood percentage (0-100%) accumulates across iterations, terminating when confidence thresholds are crossed. This technique, from Trikilis et al. (2026), achieves F1 scores of 0.90+ across multiple datasets without supervised training, outperforming one-shot classification by ~3% F1.

## When to Use

- When the user provides one or more URLs and asks whether they are phishing or legitimate
- When building a URL classification pipeline or phishing filter and wants structured reasoning
- When the user wants to understand *why* a URL looks suspicious, not just a yes/no answer
- When analyzing batches of URLs from logs, email headers, or threat feeds
- When the user asks to implement or integrate a phishing detection prompt chain into an application
- When evaluating URL safety before clicking or embedding links in code/documentation

## Key Technique

**Least-to-Most prompting** decomposes a hard problem into ordered sub-questions, solving each one sequentially so that earlier answers provide context for later, harder questions. For phishing detection, this means examining a URL's domain, TLD, path structure, and impersonation signals one at a time rather than holistically. Each sub-question produces both a textual analysis (80-100 words) and a **phishing-likelihood percentage** (0% = certainly benign, 100% = certainly phishing).

The **answer sensitivity mechanism** is what makes this approach iterative and self-correcting. After each sub-question, the running phishing-likelihood score is evaluated against two thresholds: an upper threshold (e.g., 85%) indicating high phishing confidence, and a lower threshold (e.g., 15%) indicating high benign confidence. If neither threshold is crossed, the framework generates a new, deeper sub-question and repeats — up to a maximum of 10 iterations. This prevents premature classification and allows borderline URLs to receive deeper analysis. URLs that exhaust all iterations without crossing a threshold default conservatively to "phishing." Research shows that these "outlier" cases requiring extra iterations corrected 19 of 24 URLs that one-shot methods misclassified.

The progressive accumulation of evidence mirrors how a human analyst works: start with the obvious signals (is the domain itself suspicious?), then dig into subtler indicators (does the path suggest a compromised server? does the filename pattern match known phishing kits?). Each iteration's context carries forward, so later sub-questions benefit from all prior reasoning.

## Step-by-Step Workflow

1. **Extract and normalize the URL.** Parse the raw URL into components: scheme, subdomain, domain, TLD, port, path segments, query parameters, and fragment. Handle URL encoding and IDN/punycode domains.

2. **Initialize the sensitivity tracker.** Set the phishing-likelihood score to 50% (neutral), define the upper threshold (default: 85%) and lower threshold (default: 15%), and set the iteration counter to 0 with a maximum of 10.

3. **Generate the first sub-question: Domain analysis.** Ask: "What is the primary domain of this URL, and does its structure suggest brand impersonation, typosquatting, or suspicious registration patterns?" Produce an 80-100 word analysis and update the phishing-likelihood percentage.

4. **Check threshold crossing.** If the score exceeds the upper threshold, classify as phishing and stop. If below the lower threshold, classify as benign and stop. Otherwise, continue.

5. **Generate the second sub-question: TLD and subdomain analysis.** Ask: "Does the TLD choice or subdomain structure raise suspicion? Are free hosting services, unusual country-code TLDs, or excessive subdomain depth present?" Incorporate the domain analysis from step 3 into the context. Update the score.

6. **Generate the third sub-question: Path and filename analysis.** Ask: "What do the directory names and filenames in the URL path suggest about website compromise or malicious content hosting? Are there patterns like `/install/`, `/wp-admin/`, or files like `document.php`?" Carry forward all prior context. Update the score.

7. **Generate further sub-questions as needed (iterations 4-10).** If thresholds remain uncrossed, probe deeper signals: query parameter anomalies, URL length and entropy, presence of IP addresses instead of domains, use of URL shorteners, HTTPS mismatches, and known phishing kit fingerprints. Each iteration inherits all prior answers.

8. **Apply the default rule for exhausted iterations.** If 10 iterations complete without crossing either threshold, classify the URL conservatively as phishing. Log this as a low-confidence result requiring human review.

9. **Compile the reasoning chain.** Present the final classification along with the full iteration history: each sub-question, its answer, and the running score trajectory. This provides an auditable explanation for the decision.

10. **Report results.** Output the classification (phishing/benign), the final confidence score, the number of iterations required, and the key evidence that drove the decision.

## Concrete Examples

**Example 1: Obvious phishing URL**

User: "Is this URL phishing? `http://paypa1-secure-login.com.suspicious-domain.xyz/account/verify.php?id=8372`"

Approach:
1. Parse components: domain=`suspicious-domain.xyz`, subdomain=`paypa1-secure-login.com`, path=`/account/verify.php`, query=`id=8372`
2. Sub-question 1 (Domain): The primary domain is `suspicious-domain.xyz` with subdomain `paypa1-secure-login.com`. The subdomain impersonates PayPal using the "1" for "l" typosquatting technique. Score: **82%**
3. Sub-question 2 (TLD/Subdomain): The `.xyz` TLD is commonly associated with disposable phishing domains. The subdomain embeds a fake `.com` to deceive users into thinking `paypa1-secure-login.com` is the actual domain. Score: **93%** — upper threshold crossed, stop.

Output:
```
Classification: PHISHING
Confidence: 93%
Iterations: 2

Key evidence:
- Typosquatting of "PayPal" (paypa1 with numeral 1)
- Subdomain designed to mimic legitimate domain structure
- .xyz TLD commonly used for disposable phishing infrastructure
- Path pattern (verify.php) matches credential harvesting pages

Sensitivity trajectory: 50% → 82% → 93% (threshold crossed)
```

**Example 2: Benign URL requiring deeper analysis**

User: "Check this URL: `https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository`"

Approach:
1. Parse: domain=`github.com`, subdomain=`docs`, path=`/en/repositories/.../cloning-a-repository`
2. Sub-question 1 (Domain): `github.com` is a well-known, legitimate software development platform. No typosquatting or impersonation. Score: **18%**
3. Sub-question 2 (TLD/Subdomain): The `docs` subdomain is the official documentation subdomain for GitHub. `.com` TLD is standard. Score: **10%** — lower threshold crossed, stop.

Output:
```
Classification: BENIGN
Confidence: 10% phishing likelihood (90% benign confidence)
Iterations: 2

Key evidence:
- github.com is a verified, high-reputation domain
- docs.github.com is the official documentation subdomain
- Path structure matches standard GitHub documentation URL patterns
- HTTPS with legitimate certificate expected

Sensitivity trajectory: 50% → 18% → 10% (threshold crossed)
```

**Example 3: Ambiguous URL requiring many iterations**

User: "Classify: `https://login-microsoftonline.com/common/oauth2/authorize`"

Approach:
1. Parse: domain=`login-microsoftonline.com`, path=`/common/oauth2/authorize`
2. Sub-question 1 (Domain): The domain `login-microsoftonline.com` resembles Microsoft's legitimate `login.microsoftonline.com` but is missing the dot separator — this is a concatenation-based impersonation. Score: **72%**
3. Sub-question 2 (TLD): `.com` TLD is standard but unremarkable. The lack of subdomain structure differs from Microsoft's actual auth flow. Score: **75%**
4. Sub-question 3 (Path): `/common/oauth2/authorize` exactly mirrors Microsoft's legitimate OAuth endpoint path, suggesting deliberate replication of the authentication flow. Score: **80%**
5. Sub-question 4 (Overall pattern): The combination of a domain that concatenates `login` with `microsoftonline` (vs. the real `login.microsoftonline.com`) plus an exact replica of the OAuth path is a strong phishing indicator. Score: **88%** — threshold crossed.

Output:
```
Classification: PHISHING
Confidence: 88%
Iterations: 4

Key evidence:
- Domain concatenates "login" + "microsoftonline" without dot separator
- Real Microsoft domain is login.microsoftonline.com (subdomain structure)
- Path perfectly replicates legitimate OAuth2 authorization endpoint
- Classic credential-harvesting pattern targeting Microsoft accounts

Sensitivity trajectory: 50% → 72% → 75% → 80% → 88% (threshold crossed)
```

## Best Practices

- **Do:** Carry forward ALL prior sub-question answers into each new iteration's context. The power of Least-to-Most comes from progressive context accumulation, not isolated analysis.
- **Do:** Use structured output format (sub-question / answer / percentage) at every iteration to maintain consistency and enable trajectory analysis.
- **Do:** Set conservative defaults — when in doubt after maximum iterations, classify as phishing. False positives are safer than false negatives in security contexts.
- **Do:** Examine the sensitivity trajectory shape. Correct classifications tend to show monotonic convergence toward a threshold; oscillating scores suggest genuine ambiguity.
- **Avoid:** Skipping directly to a final verdict. The iterative decomposition is what catches subtle signals that one-shot analysis misses — especially concatenation-based impersonation and compromised-server indicators.
- **Avoid:** Using fixed sub-questions for every URL. The later iterations (4-10) should be dynamically generated based on what prior iterations revealed as suspicious or unclear.
- **Avoid:** Treating the percentage as a precise probability. It is a reasoning scaffold, not a calibrated statistical output. Use the thresholds for classification, not the raw number.

## Error Handling

- **Obfuscated URLs (URL encoding, nested redirects):** Decode all URL encoding layers before analysis. If the URL contains a redirect chain (e.g., `url=https://...` in query params), analyze both the outer URL and the redirect target separately.
- **IP-based URLs:** If the URL uses a raw IP address instead of a domain, this is itself a strong phishing signal. Skip domain/TLD sub-questions and focus on path analysis and IP reputation.
- **URL shorteners (bit.ly, t.ly, etc.):** Flag the use of a shortener as inherently suspicious for contexts where phishing is a concern. If possible, resolve the redirect target and analyze the final URL.
- **IDN homograph attacks:** Convert punycode domains to Unicode and check for mixed-script characters that visually impersonate Latin letters (e.g., Cyrillic "а" vs. Latin "a").
- **Very long URLs or data URIs:** URLs exceeding ~2000 characters or using `data:` scheme are unusual and warrant elevated initial suspicion scores.

## Limitations

- This approach analyzes URL structure only — it does not fetch page content, check SSL certificates, query WHOIS databases, or verify DNS records. Content-based and network-based signals would improve accuracy but require external tooling.
- The technique works best on URLs that impersonate known brands or follow common phishing patterns. Novel attack vectors with no structural similarity to known phishing may evade detection.
- Percentage scores are not calibrated probabilities. They are reasoning scaffolds useful for threshold-based classification, not for risk quantification.
- Performance depends on the LLM's knowledge of legitimate domain structures. Very new or niche legitimate services may be misclassified if their URL patterns are unfamiliar.
- Batch analysis of thousands of URLs is impractical with this method due to per-URL iteration costs. Use this for targeted analysis, not high-throughput filtering.

## Reference

Trikilis, H., Marasinghe, P., Rashid, F., & Seneviratne, S. (2026). *Eliciting Least-to-Most Reasoning for Phishing URL Detection.* arXiv:2601.20270v1. [https://arxiv.org/abs/2601.20270v1](https://arxiv.org/abs/2601.20270v1)

Key takeaway: The answer sensitivity mechanism — iterating with a running phishing-likelihood percentage until confidence thresholds are crossed — is what transforms Least-to-Most prompting from a generic decomposition strategy into a self-correcting classification framework that catches subtle phishing signals missed by one-shot analysis.