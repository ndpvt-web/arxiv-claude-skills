---
name: "villain-at-averimatec-verifying"
description: "Build multi-agent fact-checking pipelines that verify image-text claims through modality-specific analysis, cross-modal reasoning, and structured QA generation. Use when the user says 'verify this claim with evidence', 'fact-check this image and caption', 'build a multi-agent verification pipeline', 'check if this image matches the text', 'detect misinformation in multimodal content', or 'create an evidence-based claim verifier'."
---

# VILLAIN: Multi-Agent Collaboration for Image-Text Claim Verification

This skill enables Claude to design and implement multi-agent pipelines that verify claims involving both images and text. Based on the VILLAIN system (ranked 1st on the AVerImaTeC shared task across all metrics), the approach decomposes fact-checking into four sequential stages handled by five specialized agents: evidence retrieval, modality-specific analysis, QA pair generation, and verdict prediction. The key innovation is using parallel modality-specific agents (text-text, image-text, and cross-modal) that each produce structured analysis reports, which are then synthesized into question-answer pairs before a final verdict agent renders judgment. This architecture prevents any single agent from being overwhelmed by heterogeneous evidence and ensures systematic coverage of both visual and textual dimensions.

## When to Use

- When the user asks to verify whether an image matches its accompanying text claim (e.g., "Is this photo really from the event described in the caption?")
- When building a fact-checking or misinformation detection system that must handle both images and text
- When the user wants to design a multi-agent pipeline where agents specialize by modality (text-only analysis, image-derived text analysis, cross-modal reconciliation)
- When implementing evidence retrieval systems that combine text embedding search with reverse image search results
- When the user needs structured QA-pair generation as an intermediate reasoning step before classification
- When building any verification workflow that requires aggregating evidence from multiple heterogeneous sources and producing an auditable verdict with justification

## Key Technique

VILLAIN's core insight is that multimodal fact-checking benefits from **decomposing the verification task by evidence modality before synthesizing across modalities**. Rather than feeding all evidence into a single model, VILLAIN routes text-sourced evidence and image-sourced evidence to separate specialist agents, then uses a cross-modal agent to reconcile conflicts. This division of labor prevents context window pollution (where irrelevant evidence from one modality drowns out signals from another) and produces focused analysis reports that downstream agents can reason over more effectively.

The pipeline operates in four stages. First, **evidence retrieval** populates three knowledge stores: web text matched to the claim text, web text matched via reverse image search, and web images matched to the claim image. Retrieval uses dense embeddings (mxbai-embed-large-v1) with a retrieve-then-rerank strategy (top-100 candidates reranked to top-10). Second, three **analysis agents** run in parallel: the Text-Text agent examines claim-text vs. text evidence, the Image-Text agent examines reverse-image-search evidence against the claim, and the Cross-Modal agent reconciles inconsistencies across all evidence types. Third, a **QA Generation agent** iteratively produces question-answer pairs (4 iterations of 5 pairs each, conditioned on previously generated pairs to reduce redundancy), using BM25-retrieved few-shot examples for format guidance. Finally, the **Verdict Prediction agent** selects the top-10 most relevant QA pairs and produces a verdict label, justification, and supporting evidence subset.

The iterative, conditioned QA generation is particularly important: by generating QA pairs in rounds where each round sees previous output, the system avoids redundant questions and progressively covers edge cases. This structured intermediate representation (QA pairs) is more tractable for the verdict agent than raw evidence dumps.

## Step-by-Step Workflow

1. **Parse the input claim**: Extract the textual claim and associated image(s). Identify named entities, dates, locations, and key factual assertions in the text. Describe the image content systematically (objects, people, setting, text overlays, visual context).

2. **Build three evidence retrieval channels**:
   - **Text-text channel**: Embed the textual claim using a dense embedding model, retrieve top-100 candidate text passages from a knowledge store or web search, rerank to top-10 using a cross-encoder reranker.
   - **Image-text channel**: Perform reverse image search (e.g., Google Lens or similar) to find pages where the image appears, extract text from those pages, chunk into ~2048-character segments with adjacent context.
   - **Image-image channel**: Use a multimodal embedding model to retrieve visually similar images (top-5 from text query, top-1 per claim image).

3. **Run the Text-Text Analysis Agent**: Prompt a VLM with the original claim text, the claim image, and the top-10 text-retrieved passages. Instruct it to produce a structured report identifying: (a) which facts in the claim are supported by evidence, (b) which are contradicted, (c) which lack evidence, and (d) notable context the evidence provides that the claim omits.

4. **Run the Image-Text Analysis Agent in parallel**: Prompt a VLM with the claim image, claim text, and the reverse-image-search text evidence. Instruct it to report: (a) whether the image has appeared in different contexts, (b) whether text from image-source pages contradicts the claim, (c) any metadata about the image's origin (date, source, original caption).

5. **Run the Cross-Modal Analysis Agent in parallel**: Prompt a VLM with the claim, claim image, text evidence summaries, and retrieved images. Instruct it to reconcile inconsistencies between the text-text and image-text findings, flag cases where text evidence says one thing but image evidence suggests another, and produce a unified conflict/agreement assessment.

6. **Generate QA pairs iteratively**: Feed all three analysis reports to a QA Generation agent. Generate 5 question-answer pairs per iteration across 4 iterations (20 total). In each iteration, include the previously generated pairs in the prompt context so the agent avoids redundancy and covers new angles. Use BM25 retrieval over training examples to supply 3 few-shot demonstrations of well-formed QA pairs.

7. **Select the most relevant QA pairs**: From the 20 generated pairs, use embedding similarity or the VLM itself to rank and select the top-10 QA pairs most relevant to the claim's core assertions.

8. **Predict the verdict**: Prompt the Verdict Prediction agent with the original claim text, claim image, and the top-10 QA pairs. Instruct it to output: (a) a verdict label (Supported, Refuted, Not Enough Evidence, Conflicting Evidence/Cherry-picking), (b) a natural-language justification citing specific QA pairs, and (c) the subset of QA pairs that most influenced the decision.

9. **Format the auditable output**: Structure the final response with the verdict, justification, supporting evidence links, and the full QA pair trail so that each reasoning step is traceable.

## Concrete Examples

**Example 1: Verifying a viral social media claim**

User: "I have an image of a flooded street with the caption 'Massive flooding in downtown Austin, Texas, January 2026.' Can you verify this claim?"

Approach:
1. Parse claim: Location (Austin, TX), event (flooding), time (January 2026), visual content (flooded street).
2. Text-text retrieval: Search for "flooding Austin Texas January 2026" in news archives and knowledge bases. Retrieve top-10 relevant passages.
3. Image-text retrieval: Reverse image search the flood photo. Find that the image appears on a 2024 Houston Chronicle article about Houston flooding.
4. Text-Text Agent report: "News sources confirm minor rainfall in Austin in January 2026 but no reports of 'massive flooding.' No corroborating reports from KXAN, Austin American-Statesman, or NWS Austin."
5. Image-Text Agent report: "The image was published in a Houston Chronicle article dated May 2024, captioned 'Flooding in Houston's Meyerland neighborhood.' The image does not originate from Austin."
6. Cross-Modal Agent report: "Text evidence and image evidence are inconsistent with the claim. The image is from Houston 2024, not Austin 2026. The textual claim of 'massive flooding' in Austin is unsupported by any news source."
7. QA generation (sample pairs):
   - Q: "Was there massive flooding in downtown Austin in January 2026?" A: "No major flooding events were reported in Austin in January 2026 according to NWS and local news."
   - Q: "Where does the image originate?" A: "The image was published by the Houston Chronicle in May 2024 depicting flooding in Houston's Meyerland neighborhood."
8. Verdict: **Refuted** — The image is misattributed (Houston 2024, not Austin 2026) and no corroborating reports of massive flooding in Austin exist for the claimed period.

**Example 2: Building a verification pipeline in code**

User: "Help me build a Python pipeline that implements the VILLAIN multi-agent approach for fact-checking image-text claims."

Approach:
1. Define the pipeline stages as separate modules:

```python
# pipeline.py — VILLAIN-style multi-agent fact-checking pipeline

from dataclasses import dataclass

@dataclass
class Claim:
    text: str
    image_path: str

@dataclass
class EvidenceBundle:
    text_text_passages: list[str]      # From text-based retrieval
    image_text_passages: list[str]     # From reverse image search
    similar_images: list[str]          # Paths/URLs of visually similar images

@dataclass
class AnalysisReport:
    agent_name: str
    supported_facts: list[str]
    contradicted_facts: list[str]
    missing_evidence: list[str]
    notes: str

@dataclass
class QAPair:
    question: str
    answer: str

@dataclass
class Verdict:
    label: str  # "Supported" | "Refuted" | "Not Enough Evidence" | "Conflicting"
    justification: str
    key_qa_pairs: list[QAPair]
```

2. Implement the evidence retrieval stage:

```python
def retrieve_evidence(claim: Claim, knowledge_store) -> EvidenceBundle:
    # Text-text: embed claim, retrieve top-100, rerank to top-10
    text_candidates = knowledge_store.embed_search(claim.text, top_k=100)
    text_passages = rerank(claim.text, text_candidates, top_k=10)

    # Image-text: reverse image search, extract page text, chunk
    image_source_pages = reverse_image_search(claim.image_path)
    image_text_chunks = []
    for page in image_source_pages:
        chunks = chunk_text(page.text, size=2048, overlap_adjacent=True)
        image_text_chunks.extend(chunks)
    image_passages = rerank(claim.text, image_text_chunks, top_k=10)

    # Image-image: multimodal embedding similarity
    similar = knowledge_store.visual_search(claim.image_path, top_k=5)

    return EvidenceBundle(text_passages, image_passages, similar)
```

3. Implement the three parallel analysis agents:

```python
import asyncio

async def run_analysis_agents(claim: Claim, evidence: EvidenceBundle) -> list[AnalysisReport]:
    tasks = [
        text_text_agent(claim, evidence.text_text_passages),
        image_text_agent(claim, evidence.image_text_passages),
        cross_modal_agent(claim, evidence),
    ]
    return await asyncio.gather(*tasks)

async def text_text_agent(claim: Claim, passages: list[str]) -> AnalysisReport:
    prompt = f"""Analyze the following claim against the text evidence.
Claim: {claim.text}
Evidence passages: {passages}

Produce a structured report with:
- supported_facts: facts in the claim backed by evidence
- contradicted_facts: facts contradicted by evidence
- missing_evidence: claims with no supporting or contradicting evidence
- notes: additional context the evidence reveals"""
    response = await call_vlm(prompt, image=claim.image_path)
    return parse_analysis_report("Text-Text", response)
```

4. Implement iterative QA generation with redundancy conditioning:

```python
async def generate_qa_pairs(
    claim: Claim,
    reports: list[AnalysisReport],
    few_shot_examples: list[QAPair],
    iterations: int = 4,
    pairs_per_iter: int = 5,
) -> list[QAPair]:
    all_pairs = []
    for i in range(iterations):
        prompt = f"""Given this claim and analysis reports, generate {pairs_per_iter}
question-answer pairs that verify or refute the claim.

Claim: {claim.text}
Reports: {format_reports(reports)}
Few-shot examples: {format_qa(few_shot_examples)}
Previously generated (DO NOT repeat): {format_qa(all_pairs)}

Generate {pairs_per_iter} NEW, non-redundant QA pairs."""
        response = await call_vlm(prompt, image=claim.image_path)
        all_pairs.extend(parse_qa_pairs(response))
    return all_pairs
```

5. Implement verdict prediction with QA selection:

```python
async def predict_verdict(claim: Claim, qa_pairs: list[QAPair]) -> Verdict:
    top_pairs = select_top_qa(claim.text, qa_pairs, top_k=10)
    prompt = f"""Based on the claim and these question-answer pairs, predict a verdict.

Claim: {claim.text}
QA Pairs: {format_qa(top_pairs)}

Output:
- label: one of [Supported, Refuted, Not Enough Evidence, Conflicting Evidence/Cherry-picking]
- justification: 2-3 sentence explanation citing specific QA pairs
- key_qa_pairs: indices of the most influential QA pairs"""
    response = await call_vlm(prompt, image=claim.image_path)
    return parse_verdict(response, top_pairs)
```

**Example 3: Adapting the approach for document verification**

User: "I want to verify whether a screenshot of a news article is authentic and whether the headline matches the article content."

Approach:
1. Treat the screenshot as the "image" and the headline as the "text claim."
2. Text-text retrieval: Search for the headline text across news databases and fact-checking sites.
3. Image-text retrieval: Reverse image search the screenshot to find the original article URL. Extract the actual article text from that URL.
4. Text-Text Agent: Compare headline claims against retrieved news coverage from other sources.
5. Image-Text Agent: Compare the screenshot content against the actual article found via reverse search. Check for manipulated text, altered headlines, or cropped context.
6. Cross-Modal Agent: Reconcile whether the screenshot matches the actual published article and whether other sources corroborate the headline's claims.
7. Generate QA pairs: "Does the original article contain this headline?" / "Has the article text been altered in the screenshot?" / "Do other sources report the same claim?"
8. Verdict: Supported (authentic and accurate), Refuted (manipulated or misattributed), or Conflicting (real screenshot but misleading headline).

## Best Practices

- **Do** run analysis agents in parallel — the Text-Text, Image-Text, and Cross-Modal agents are independent and benefit from concurrent execution.
- **Do** condition each QA generation iteration on previously generated pairs. This is the single most important technique for avoiding redundant questions and ensuring coverage.
- **Do** use a retrieve-then-rerank strategy (broad recall first, precision reranking second) rather than relying on a single retrieval pass.
- **Do** chunk long evidence texts into ~2048-character segments with adjacent chunk overlap so that analysis agents receive coherent passages rather than truncated fragments.
- **Do** include the claim image in every agent prompt, even text-focused ones — the VLM can catch visual details that inform textual analysis.
- **Avoid** feeding all evidence into a single agent prompt. The modality-specific decomposition is what prevents context pollution and improves both recall and precision.
- **Avoid** generating all QA pairs in a single pass. The iterative approach with conditioning is measurably better (VILLAIN's ablation showed evidence analysis agents improved Evid-Eval by 0.040–0.067).
- **Avoid** skipping the analysis report stage and jumping straight to QA generation. The intermediate reports give the QA agent structured findings to work from rather than raw evidence.

## Error Handling

- **Empty evidence retrieval**: If text-text or image-text retrieval returns no results, the corresponding analysis agent should explicitly state "No evidence found for this modality" in its report. The Cross-Modal agent should flag this as a gap. The QA agent should generate questions acknowledging the evidence gap rather than hallucinating answers.
- **Conflicting analysis reports**: When the Text-Text and Image-Text agents reach opposite conclusions, the Cross-Modal agent must explicitly enumerate the conflicts. The verdict agent should lean toward "Conflicting Evidence/Cherry-picking" rather than forcing a binary Supported/Refuted label.
- **Reverse image search failures**: If reverse image search returns no results, fall back to describing the image content via the VLM and using that description for text-based retrieval. Note the degraded evidence quality in the analysis report.
- **QA generation producing duplicates despite conditioning**: If the iterative QA generation still produces near-duplicate questions, apply deduplication via embedding similarity (threshold > 0.9) before the selection step.
- **Context window overflow**: If the combined evidence exceeds the model's context window, prioritize higher-ranked evidence and truncate lower-ranked passages. Never silently drop evidence — log which passages were excluded.

## Limitations

- **Requires multimodal model access**: The pipeline depends on vision-language models for every agent. Text-only LLMs cannot serve as the Image-Text or Cross-Modal agents.
- **Evidence retrieval quality is a bottleneck**: The entire pipeline is only as good as its evidence. If the knowledge store is incomplete or web retrieval is blocked, downstream agents cannot compensate.
- **Not real-time**: The four-stage sequential pipeline with iterative QA generation involves multiple LLM calls (minimum 6: 3 analysis + 4 QA iterations + 1 verdict, often more). This is a batch verification approach, not suitable for real-time content moderation.
- **Language and cultural context**: The system was evaluated on English-language claims. Verification of claims in other languages requires adapted retrieval and potentially different knowledge sources.
- **Cannot verify purely subjective claims**: The pipeline is designed for factual verification. Claims that are opinions, predictions, or inherently unfalsifiable will produce "Not Enough Evidence" verdicts regardless of evidence availability.
- **Reverse image search dependency**: The Image-Text channel relies heavily on reverse image search services, which may not find matches for AI-generated images or heavily edited photos.

## Reference

**Paper**: [VILLAIN at AVerImaTeC: Verifying Image-Text Claims via Multi-Agent Collaboration](https://arxiv.org/abs/2602.04587v1) — Jung et al., 2026. Focus on Section 3 (System Description) for the four-stage pipeline architecture and Section 4 (Experiments) for ablation results showing the contribution of each agent. Source code: [github.com/ssu-humane/VILLAIN](https://github.com/ssu-humane/VILLAIN).