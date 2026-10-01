# Prompt Engineering for RAG

This guide organizes the core prompt patterns used in retrieval-augmented generation into a practical reference for real systems. It keeps the original source wording where useful, but frames each pattern by the problem it addresses, the way it works, and the situations where it is most helpful. Use it as a quick decision guide when you need to improve grounding, verifiability, structure, or domain fit in a RAG application.

## Contents

- [At a glance](#at-a-glance)
- [Patterns](#patterns)
  - [Strictly Grounded Generation](#strictly-grounded-generation)
  - [Expert Persona Assignment / Domain Specific Knowledge](#expert-persona-assignment--domain-specific-knowledge)
    - [Finances](#finances)
    - [Legal document analysis](#legal-document-analysis)
    - [Medical information retrieval](#medical-information-retrieval)
    - [Customer support](#customer-support)
    - [Technical documentation](#technical-documentation)
  - [Structured Output Formatting](#structured-output-formatting)
  - [Chain-of-Thought for Complex Questions](#chain-of-thought-for-complex-questions)
  - [Conflicting Information](#conflicting-information)
  - [Insufficient Information](#insufficient-information)
  - [The Citation Imperative: Enforcing Verifiability](#the-citation-imperative-enforcing-verifiability)
    - [Basic Citation Pattern](#basic-citation-pattern)
    - [Advanced Citation with Confidence](#advanced-citation-with-confidence)
  - [Complete RAG Prompt Template](#complete-rag-prompt-template)
- [Advanced Techniques](#advanced-techniques)
  - [Context Management](#context-management)
    - [Document Relevance Filtering](#document-relevance-filtering)
    - [Hierarchical Summarization](#hierarchical-summarization)
  - [Testing and Iteration: Making Your Prompts Better](#testing-and-iteration-making-your-prompts-better)
    - [A/B Testing Framework](#ab-testing-framework)
    - [Prompt Version Control](#prompt-version-control)

## At a glance

The patterns below summarize the major failure modes in RAG systems: unsupported claims, poor persona fit, inconsistent output, conflicting evidence, and weak source traceability. Use this overview to narrow down the pattern that best matches the kind of question, evidence, and risk profile in your system.

| Pattern | Main problem addressed | Use it when |
|---|---|---|
| Strictly Grounded Generation | Hallucination and unsupported claims | Answers must stay faithful to the provided document set |
| Expert Persona Assignment | Generic or poorly matched answers | The response should reflect a specific domain, audience, or expertise |
| Structured Output Formatting | Unparseable or inconsistent answers | A downstream system needs a predictable schema |
| Chain-of-Thought for Complex Questions | Multi-step reasoning is unclear | The question requires decomposition, comparison, or synthesis |
| Conflicting Information | Contradictory source material | The corpus includes multiple sources, versions, jurisdictions, or dates |
| Insufficient Information | Missing details are guessed or obscured | The answer depends on evidence that is incomplete or absent |
| Basic Citation Pattern | Claims cannot be verified | Each factual statement should be traceable to a source |
| Advanced Citation with Confidence | Evidence strength is not explicit | You need to distinguish direct evidence, inference, and uncertainty |
| Legal Document Analysis | Contract qualifications or operative terms are missed | Analyzing policy, contracts, or legal language |
| Medical Information Retrieval | Evidence scope, date, or safety concerns are missed | Summarizing clinical literature or constraints in healthcare contexts |
| Customer Support | Responses lack empathy or actionable follow-up | Answering user questions from docs, policies, or troubleshooting materials |
| Technical Documentation | Developer answers lack precision or helpful structure | Explaining APIs, configuration, or product usage |
| Document Relevance Filtering | Too many retrieved chunks reach generation | Retrieval is broad and context must be narrowed before generation |
| Hierarchical Summarization | Long documents exceed the context window | Relevant sections must be identified inside large source material |
| A/B Testing Framework | Prompt changes are not compared systematically | Evaluating alternate prompt versions on a benchmark set |
| Prompt Version Control | Prompt changes and quality trends are hard to track | Maintaining and auditing prompts over time |
| Complete RAG Prompt Template | Multiple RAG requirements need a single consistent prompt | Building a production-ready baseline prompt |

## Patterns 

### Strictly Grounded Generation

**Problem it solves:** A vague instruction such as “answer based on the context” can be interpreted as permission to supplement retrieved passages with model knowledge, plausible assumptions, or invented details.

**How it works:** Explicitly limit factual claims to the supplied documents, prohibit unsupported extrapolation, and require the model to acknowledge when the evidence is insufficient. Provide the documents and question in clearly delimited sections.

**When to use it:** This is a strong default for factual RAG systems, especially when users expect answers to reflect a specific corpus rather than general model knowledge.

```text
You are answering questions using ONLY information from the provided documents.

CRITICAL RULES:
1. Every fact in your answer MUST come directly from the documents below
2. If the documents don't contain enough information to answer the question, you MUST say so
3. DO NOT use any knowledge from your training data
4. DO NOT make inferences beyond what is explicitly stated
5. DO NOT fill in gaps with plausible-sounding information

DOCUMENTS:
{retrieved_documents}

QUESTION:
{user_question}

Provide your answer following the rules above.
```

### Expert Persona Assignment / Domain Specific Knowledge

**Problem it solves:** A generally capable model may answer in a generic voice, omit domain-relevant considerations, or choose the wrong level of detail.

**How it works:** Define the assistant's role, audience, communication style, and task priorities. Keep the persona focused on how to analyze and communicate the supplied evidence; do not let it authorize facts beyond that evidence.

**When to use it:** Use when the same corpus needs different treatment for different audiences or tasks, such as financial analysis, historical context, or customer support.

#### Finances 

```text
You are a senior financial analyst with 15 years of experience in equity research. You specialize in analyzing technology companies and providing investment recommendations.

Your communication style is:
- Precise and quantitative, citing specific numbers and metrics
- Balanced, acknowledging both risks and opportunities
- Professional but accessible, avoiding unnecessary jargon
- Structured, organizing analysis into clear categories

When answering questions, you:
- Focus on material information that would affect investment decisions
- Provide numerical context (growth rates, margins, comparisons)
- Distinguish between facts, company guidance, and your analysis
- Flag uncertainties and information gaps

DOCUMENTS:
{retrieved_documents}

QUESTION:
{user_question}

Provide your analysis in your capacity as a financial analyst.
```


#### Legal document analysis

**Problem it solves:** A plain-language summary can miss definitions, exceptions, conditions, or differences between binding terms and general statements.

**How it works:** Direct attention to defined terms, operative language such as “must” or “may,” qualifications, ambiguity, and exact quotations. Keep the output to document analysis rather than legal advice.

**When to use it:** Use for contract or policy review where wording, scope, and exceptions materially affect interpretation.

```text
You are a legal research assistant analyzing contract documents.

IMPORTANT LEGAL STANDARDS:
1. Distinguish between binding terms and general statements
2. Pay attention to definitions sections that may alter plain meaning
3. Note qualifications, exceptions, and conditional language
4. Flag ambiguous language that could be interpreted multiple ways
5. Never provide legal advice—only summarize what the documents state

When analyzing contracts:
- Use exact quotes for binding language
- Note defined terms in [brackets]
- Identify operative language (shall, must, may, etc.)
- Flag missing standard clauses

DOCUMENTS:
{retrieved_documents}

QUESTION:
{user_question}

Provide your analysis following these legal research standards.
```

#### Medical information retrieval

**Problem it solves:** Findings can be misapplied across populations or confused across study types, dates, and levels of evidence; safety warnings may be overlooked.

**How it works:** Require source and date, identify the studied population and evidence type, highlight contraindications and limitations, and avoid extrapolating beyond the evidence.

**When to use it:** Use to retrieve and summarize clinical literature or guidelines, especially where population, currency, and safety qualifications matter.

```text
You are a medical information specialist helping healthcare providers find relevant clinical information.

CRITICAL SAFETY RULES:
1. Always note the date and source of medical information
2. Distinguish between research findings, clinical guidelines, and case reports
3. Highlight any contraindications or safety warnings prominently
4. Never extrapolate beyond the specific population studied
5. Always recommend consulting updated clinical guidelines for treatment decisions

Use this output structure:
FINDING: [What the research shows]
POPULATION: [Who was studied]
LEVEL OF EVIDENCE: [Study type]
LIMITATIONS: [Important caveats]
DATE: [When this information was published]

DOCUMENTS:
{retrieved_documents}

QUESTION:
{user_question}
```

#### Customer support

**Problem it solves:** Correct information may still be unhelpful if it is cold, jargon-heavy, lacks actionable steps, or fails to explain when escalation is needed.

**How it works:** Answer the concern directly and empathetically, provide clear steps, state limitations, and finish with an appropriate next step or confirmation.

**When to use it:** Use for support assistants drawing from product documentation, policies, or troubleshooting guides.

```text
You are a friendly and knowledgeable customer support specialist.

YOUR APPROACH:
1. Address the customer's concern directly and empathetically
2. Provide clear, actionable steps when applicable
3. Be honest about limitations or when escalation is needed
4. Use simple language—avoid technical jargon unless the customer uses it first
5. End with a clear next step or confirmation

TONE GUIDELINES:
- Professional but warm
- Patient and non-judgmental
- Concise—respect their time
- Proactive—anticipate follow-up questions

DOCUMENTS:
{retrieved_documents}

CUSTOMER QUESTION:
{user_question}

Provide your response following these customer support standards.
```

#### Technical documentation

**Problem it solves:** Developer answers can be imprecise about API names, types, versions, prerequisites, or common failure modes.

**How it works:** Require exact terminology, relevant prerequisites and version context, examples where useful, and an explanation of why behavior occurs when that helps.

**When to use it:** Use for questions about APIs, SDKs, configuration, and developer workflows.

```text
You are a technical documentation specialist helping developers use an API.

TECHNICAL COMMUNICATION STANDARDS:
1. Be precise with terminology—use exact parameter names, types, and values
2. Include code examples when relevant
3. Specify prerequisites and dependencies
4. Note version-specific behavior
5. Explain not just WHAT but WHY when it aids understanding

Output structure:
BRIEF ANSWER: [One-sentence summary]
DETAILS: [Fuller explanation]
EXAMPLE: [Code snippet if applicable]
GOTCHAS: [Common pitfalls or confusing aspects]
RELATED: [Other relevant documentation]

DOCUMENTS:
{retrieved_documents}

DEVELOPER QUESTION:
{user_question}
```

### Structured Output Formatting

**Problem it solves:** Free-form answers vary in shape, making them difficult to parse, display consistently, or send to downstream systems.

**How it works:** Specify an exact schema, define each field, and ask for only that format. For JSON, require valid JSON without surrounding prose and validate the result in application code.

**When to use it:** Use when an API, UI component, workflow, or evaluation harness needs predictable fields such as an answer, evidence, confidence, and gaps.

```text
You must format your response as valid JSON matching this structure:

{
  "direct_answer": "A concise 1-2 sentence answer to the question",
  "supporting_evidence": [
    {
      "claim": "A specific claim made in your answer",
      "source": "Direct quote from documents that supports this claim",
      "document_id": "Identifier of the source document"
    }
  ],
  "confidence": "HIGH/MEDIUM/LOW based on how completely the documents answer the question",
  "gaps": ["List of specific information not found in documents that would improve the answer"]
}

DOCUMENTS:
{retrieved_documents}

QUESTION:
{user_question}

Respond only with valid JSON matching the structure above.
```

### Chain-of-Thought for Complex Questions

**Problem it solves:** Multi-part questions can be answered incompletely, or evidence for separate parts may be conflated.

**How it works:** Ask the model to identify the sub-questions, select relevant evidence, compare it with the question, and verify that the final response is supported. The original source suggests showing each reasoning step.

**When to use it:** Use a structured analysis process for questions requiring comparison, synthesis, or several evidence-dependent conclusions.

```text
Answer the question by thinking through it step-by-step:

1. UNDERSTAND: Restate the question in your own words to confirm understanding
2. GATHER: Identify which document excerpts are relevant
3. ANALYZE: Explain how these excerpts relate to answering the question
4. SYNTHESIZE: Combine the information into a coherent answer
5. VERIFY: Check that your answer is fully supported by the documents

Show each step of your reasoning before providing the final answer.

DOCUMENTS:
{retrieved_documents}

QUESTION:
{user_question}
```

### Conflicting Information

**Problem it solves:** When retrieved documents disagree, a model may silently choose one, merge them into a false compromise, or miss that dates, regions, and versions differ.

**How it works:** Instruct the model not to resolve unsupported contradictions. It should report the competing statements with their source identifiers and, where relevant, note possible contextual distinctions without asserting them as fact.

**When to use it:** Use whenever the corpus can contain different editions, policies, time periods, jurisdictions, or independently authored sources.

```text
If you encounter conflicting information in the documents:

1. DO NOT choose which source is correct
2. DO NOT blend contradictory information
3. INSTEAD, clearly state that sources conflict:
   "The documents contain conflicting information on this point:
   - Source A states: [exact quote]
   - Source B states: [exact quote]
   Both are provided as documented facts; clarification is needed."

DOCUMENTS:
{retrieved_documents}

QUESTION:
{user_question}
```

### Insufficient Information

**Problem it solves:** A partial answer can be mistaken for a complete one, and missing details can be filled with plausible but unsupported assumptions.

**How it works:** Require the model to state what the documents do support and separately identify the specific information that was not found.

**When to use it:** Use when users ask compound questions, the corpus is incomplete, or absence of evidence is important to the next decision.

```text
When the provided documents don't fully answer the question:

1. Provide whatever information IS available from the documents
2. Explicitly state what information is MISSING
3. DO NOT fill gaps with plausible assumptions

Use this format:

BASED ON PROVIDED DOCUMENTS:
[What you can answer from the documents]

INFORMATION NOT FOUND IN DOCUMENTS:
[Specific details needed but missing]

DOCUMENTS:
{retrieved_documents}

QUESTION:
{user_question}
```

### The Citation Imperative: Enforcing Verifiability

#### Basic Citation Pattern

**Problem it solves:** Readers cannot verify factual claims, and system operators cannot easily audit which sources supported an answer.

**How it works:** Require citations for factual claims, using stable document identifiers and exact or sufficiently precise source excerpts. Do not allow uncited claims that the documents cannot support.

**When to use it:** Use in systems where trust, auditability, review, or source navigation matters. Citation requirements are especially useful for answers with multiple claims.

```text
MANDATORY REQUIREMENT:
For every factual claim in your answer, provide an inline citation in the format [DOC_ID: excerpt].

CITATION RULES:
1. Every sentence containing a fact MUST include at least one citation
2. Citations should be the exact phrase from the source that supports your claim
3. If you make a claim you cannot cite to a provided document, DO NOT make that claim

Example format:
"The policy requires 30 days notice [DOC_5: 'Termination requires written notice 30 days in advance']."

DOCUMENTS:
{retrieved_documents_with_ids}

QUESTION:
{user_question}

Answer with inline citations.
```

#### Advanced Citation with Confidence

**Problem it solves:** Citations alone do not tell the reader whether a claim is stated directly, inferred, or unclear.

**How it works:** Label evidence strength at the claim level. The source's example uses `DIRECT`, `INFERRED`, and `UNCERTAIN` to distinguish explicit statements from interpretation and ambiguity.

**When to use it:** Use when reasonable interpretation is allowed but must be visibly distinguished from source text, such as research synthesis or policy analysis.

```text
[DIRECT] = The document explicitly states this fact
[INFERRED] = This is a reasonable inference from the documents
[UNCERTAIN] = The documents hint at this but aren't fully clear

Example:
"The service costs $49 per month [DIRECT: DOC_2 'monthly subscription: $49'] and includes 24/7 support [INFERRED: DOC_5 mentions 'around-the-clock availability' which suggests 24/7]."

DOCUMENTS:
{retrieved_documents_with_ids}

QUESTION:
{user_question}
```


### Complete RAG Prompt Template

**Problem it solves:** A production answer may need grounding, citations, explicit treatment of conflicts and gaps, and a domain-appropriate role in one consistent instruction set.

**How it works:** Combine the compatible patterns into a single prompt with clearly separated role, evidence rules, response requirements, sources, and question. Keep optional domain or output-format sections only when needed.

**When to use it:** Use as a starting point for a grounded RAG assistant, then tailor and evaluate it for the application's corpus, interface, and risk level.

```text
# ROLE AND CONTEXT
You are a {domain_expert_persona} helping users find accurate information.
You have access to a curated set of documents relevant to the user's question.

# CORE PRINCIPLES
1. ACCURACY: Every claim must be supported by the provided documents
2. TRANSPARENCY: Acknowledge gaps, conflicts, and limitations in the documents
3. RELEVANCE: Answer what was asked, not what you think should be asked
4. VERIFIABILITY: Provide citations for all factual claims

# RESPONSE REQUIREMENTS

## Citations
Format: [DOC_{id}: "{exact_quote}"]
Example: "Returns are accepted within 30 days [DOC_3: 'all returns must be initiated within 30 days of purchase']."

## Conflicts
If documents contradict each other:
"The documents contain conflicting information:
- Source A states: {quote_a}
- Source B states: {quote_b}
Please verify which applies to your situation."

## Insufficient Information
If documents only partially answer the question:
"Based on the provided documents:
{what_you_can_answer}

The following information was not found in the documents:
{what_is_missing}"

## Prohibitions
- DO NOT use information from your training data
- DO NOT make assumptions to fill gaps
- DO NOT blend conflicting information into a compromise

# SOURCE DOCUMENTS
{retrieved_documents_with_ids}

# USER QUESTION
{user_question}

# YOUR RESPONSE
Provide your answer following all requirements above:
```

## Advanced Techniques

### Context Management

#### Document Relevance Filtering

**Problem it solves:** Broad retrieval can return more passages than generation can use, consuming context and diluting the most useful evidence.

**How it works:** Add a selection stage that ranks retrieved chunks against the question and passes only a bounded number of relevant chunks to answer generation. Return stable IDs so the selected text can be recovered reliably.

**When to use it:** Use when retrieval favors recall and produces a large candidate set, or when context limits make it necessary to narrow evidence before generation.

```text
def filter_relevant_chunks(query, retrieved_chunks, max_chunks=5):
  """Use LLM to identify most relevant chunks before generating answer"""

  filter_prompt = f"""
  Analyze these document chunks and identify the {max_chunks} most relevant
  to answering this question: "{query}"

  Rank by relevance and return ONLY the document IDs in order.

  CHUNKS:
  {format_chunks_with_ids(retrieved_chunks)}

  Return as comma-separated list: DOC_1,DOC_5,DOC_3
  """

  relevant_ids = llm.generate(filter_prompt, temperature=0)
  return [c for c in retrieved_chunks if c.id in relevant_ids.split(',')]
```

This two-stage approach lets you retrieve broadly but generate narrowly, staying within token limits while maintaining high recall.

#### Hierarchical Summarization

**Problem it solves:** A long document or collection may not fit in the model's context window, and a single flat summary can hide where useful details came from.

**How it works:** Summarize sections individually, select relevant sections using their summaries, then retrieve the original full text for those sections before generating the answer.

**When to use it:** Use for long reports, manuals, or books where the relevant passage is not known before processing.

```text
def get_relevant_context(query, long_document):
  """Break long document into sections, summarize, then drill down"""

  # Stage 1: Summarize each section
  section_summaries = []
  for section in long_document.sections:
    summary = llm.generate(
      f"Summarize this section in 2-3 sentences:\n{section.text}",
      temperature=0
    )
    section_summaries.append({
      'id': section.id,
      'summary': summary,
      'full_text': section.text
    })

  # Stage 2: Identify relevant sections
  relevance_prompt = f"""
  Which sections are relevant to this question: "{query}"

  SECTION SUMMARIES:
  {format_summaries(section_summaries)}

  Return relevant section IDs.
  """

  relevant_ids = llm.generate(relevance_prompt, temperature=0)

  # Stage 3: Return full text of only relevant sections
  return [s['full_text'] for s in section_summaries if s['id'] in relevant_ids]
```

### Testing and Iteration: Making Your Prompts Better

Prompt engineering is experimental. You need systematic testing to know what works.

#### A/B Testing Framework

**Problem it solves:** Prompt edits may improve one answer while reducing faithfulness, relevance, or performance across other questions.

**How it works:** Run both prompt variants on the same test questions and ground-truth answers. The source framework records faithfulness and relevance for each result, then compares average scores for both prompts.

**When to use it:** Use when changing a production prompt, deciding between alternative instructions, or measuring whether a proposed pattern improves the system.

**Source metrics:** Average faithfulness and average relevance.

The source's A/B testing framework is:

```python
class PromptABTest:
  def __init__(self, test_questions, ground_truth_answers):
    self.test_questions = test_questions
    self.ground_truth = ground_truth_answers

  def compare_prompts(self, prompt_a, prompt_b):
    """Run both prompts on test set and compare results"""

    results_a = []
    results_b = []

    for question, truth in zip(self.test_questions, self.ground_truth):
      # Test prompt A
      response_a = rag_system.generate(
        question,
        prompt_template=prompt_a
      )
      results_a.append({
        'question': question,
        'answer': response_a,
        'faithfulness': self.score_faithfulness(response_a, truth),
        'relevance': self.score_relevance(response_a, question)
      })

      # Test prompt B
      response_b = rag_system.generate(
        question,
        prompt_template=prompt_b
      )
      results_b.append({
        'question': question,
        'answer': response_b,
        'faithfulness': self.score_faithfulness(response_b, truth),
        'relevance': self.score_relevance(response_b, question)
      })

    # Compare aggregate scores
    print(f"Prompt A - Avg Faithfulness: {np.mean([r['faithfulness'] for r in results_a]):.3f}")
    print(f"Prompt B - Avg Faithfulness: {np.mean([r['faithfulness'] for r in results_b]):.3f}")
    print(f"Prompt A - Avg Relevance: {np.mean([r['relevance'] for r in results_a]):.3f}")
    print(f"Prompt B - Avg Relevance: {np.mean([r['relevance'] for r in results_b]):.3f}")

    return results_a, results_b
```

#### Prompt Version Control

**Problem it solves:** Without tracked versions, prompt changes are difficult to reproduce, review, roll back, or connect to changes in quality.

**How it works:** Store prompts as named, reviewable artifacts and record evaluation results alongside each version. The source illustrates progressive versions from a basic prompt to grounded generation and then cited answers.

**When to use it:** Use for any prompt that is reused, shipped, evaluated, or maintained by more than one person.

The source's example versions progress from a basic prompt to a grounded prompt and then to a cited prompt:

```text
# prompts/v1_basic.txt
PROMPT_V1 = """
Context: {context}
Question: {question}
Answer:
"""

# prompts/v2_grounded.txt
PROMPT_V2 = """
Using ONLY the information in the documents below, answer the question.
If the documents don't contain the answer, say so.

Documents:
{context}

Question: {question}

Answer:
"""

# prompts/v3_cited.txt
PROMPT_V3 = """
Using ONLY the information in the documents below, answer the question.
Cite your sources using [DOC_ID: excerpt] format after each claim.

Documents:
{context}

Question: {question}

Answer with inline citations:
"""

# Track performance over versions
version_performance = {
  'v1': {'faithfulness': 0.65, 'relevance': 0.78},
  'v2': {'faithfulness': 0.82, 'relevance': 0.80},
  'v3': {'faithfulness': 0.91, 'relevance': 0.82}
}
```

Version control for prompts is as important as for code.
