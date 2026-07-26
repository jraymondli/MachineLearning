Yes—that workflow should be implemented primarily in **code**, with the LLM responsible for bounded reasoning steps inside it. The LLM should not be expected to reliably execute the entire workflow merely because it appears in the system prompt.

A practical separation is:

```text
Code owns:
- Workflow state and stage transitions
- Plan-schema validation
- Hard-constraint enforcement
- Candidate deduplication
- Pagination and search completeness
- Evidence storage
- Deterministic scoring inputs
- Stopping criteria
- Verification requirements

LLM owns:
- Translating the query into a structured plan
- Expanding ambiguous concepts
- Extracting normalized claims from evidence
- Assessing semantic relevance
- Proposing the best candidate
- Explaining the result
```

Your current generic `ToolLoop` can remain as a lower-level primitive, but you would place a higher-level constrained-search orchestrator around it:

```go
func RunConstrainedSearch(
    ctx context.Context,
    query string,
) (*SearchAnswer, error) {
    // 1. Produce and validate the structured plan.
    plan, err := generateConstraintPlan(ctx, query)
    if err != nil {
        return nil, err
    }

    if err := validatePlan(plan); err != nil {
        return nil, err
    }

    state := NewSearchState(query, plan)

    // 2. Generate candidates using objective constraints.
    candidates, err := generateCandidates(ctx, plan)
    if err != nil {
        return nil, err
    }

    // 3. Enforce hard constraints deterministically.
    candidates = filterHardConstraints(candidates, plan)
    state.SetCandidates(candidates)

    // 4. Retrieve and normalize evidence per candidate.
    for _, candidate := range candidates {
        evidence, err := retrieveCandidateEvidence(
            ctx,
            candidate,
            plan,
        )
        if err != nil {
            state.RecordCandidateError(candidate.ID, err)
            continue
        }

        claims, err := extractNormalizedClaims(
            ctx,
            candidate,
            evidence,
            plan,
        )
        if err != nil {
            state.RecordCandidateError(candidate.ID, err)
            continue
        }

        state.AddEvidence(candidate.ID, evidence)
        state.AddClaims(candidate.ID, claims)
    }

    // 5. Score and rank in code.
    ranked := scoreAndRankCandidates(state)
    state.SetRanking(ranked)

    // 6. Ask the LLM to propose an answer from bounded state.
    proposal, err := proposeFinalCandidate(ctx, state)
    if err != nil {
        return nil, err
    }

    // 7. Verify support and constraint satisfaction.
    verification, err := verifyProposal(
        ctx,
        proposal,
        state,
    )
    if err != nil {
        return nil, err
    }

    // 8. Reject unsupported proposals rather than forcing an answer.
    if !verification.Accepted {
        return buildUncertainAnswer(state, verification), nil
    }

    return composeAnswer(state, proposal, verification), nil
}
```

## The structured plan

The first LLM call should return a strict schema, not prose:

```go
type ConstraintPlan struct {
    Intent      string       `json:"intent"`
    TargetCount int          `json:"target_count"`
    Constraints []Constraint `json:"constraints"`
    Concepts    []Concept    `json:"concepts"`
    Strategy    SearchPlan   `json:"strategy"`
}

type Constraint struct {
    Field       string `json:"field"`
    Operator    string `json:"operator"`
    Value       any    `json:"value"`
    Strength    string `json:"strength"` // hard or soft
    Temporal    string `json:"temporal,omitempty"`
    SourceBasis string `json:"source_basis,omitempty"`
}

type Concept struct {
    Canonical  string   `json:"canonical"`
    Expansions []string `json:"expansions"`
}
```

For the Wang query:

```json
{
  "intent": "find_person",
  "target_count": 1,
  "constraints": [
    {
      "field": "company",
      "operator": "equals",
      "value": "Moveworks",
      "strength": "hard",
      "temporal": "current"
    },
    {
      "field": "last_name",
      "operator": "equals",
      "value": "Wang",
      "strength": "hard"
    },
    {
      "field": "expertise",
      "operator": "supported_by_evidence",
      "value": "backend systems",
      "strength": "soft"
    }
  ],
  "concepts": [
    {
      "canonical": "backend systems",
      "expansions": [
        "backend services",
        "distributed systems",
        "platform engineering",
        "service architecture",
        "infrastructure",
        "scalability"
      ]
    }
  ],
  "strategy": {
    "candidate_generation": [
      "company",
      "last_name"
    ],
    "evidence_dimensions": [
      "technical ownership",
      "design authorship",
      "project leadership",
      "repeated contribution"
    ]
  }
}
```

Your validator should reject or repair plans that:

* Lack an intent.
* Have no candidate-generation constraints.
* Mark subjective concepts such as “expert” as exact hard filters.
* Use unsupported fields or operators.
* Fail to capture explicit query constraints.

## Candidate generation should be a separate stage

Do not let the general tool loop freely alternate between finding people and reading arbitrary documents.

```go
func generateCandidates(
    ctx context.Context,
    plan ConstraintPlan,
) ([]Candidate, error) {
    filters := candidateFilters(plan)

    result, err := peopleTool.Search(ctx, filters)
    if err != nil {
        return nil, err
    }

    candidates := result.Items

    for result.HasMore {
        result, err = peopleTool.SearchNext(ctx, result.NextCursor)
        if err != nil {
            return nil, err
        }

        candidates = append(candidates, result.Items...)
    }

    return deduplicateCandidates(candidates), nil
}
```

This gives code control over pagination and completeness.

## Evidence retrieval should be candidate-scoped

After identifying the Wang candidates, retrieve evidence for each one independently:

```go
func retrieveCandidateEvidence(
    ctx context.Context,
    candidate Candidate,
    plan ConstraintPlan,
) ([]Evidence, error) {
    queries := buildEvidenceQueries(candidate, plan)

    var evidence []Evidence

    for _, query := range queries {
        results, err := searchTool.Search(ctx, query)
        if err != nil {
            continue
        }

        evidence = append(evidence, results.Items...)
    }

    return deduplicateEvidence(evidence), nil
}
```

Possible queries:

```text
"Full Name" backend systems
"Full Name" distributed systems
"Full Name" platform infrastructure
"Full Name" service architecture
"Full Name" design author
"Full Name" technical lead
```

The candidate ID or person filter should be used whenever the source supports it. Name-only evidence searches are vulnerable to collisions.

## Normalize evidence into claims

Raw documents should not flow directly into scoring. Ask the LLM to produce normalized claims:

```go
type Claim struct {
    CandidateID    string   `json:"candidate_id"`
    Predicate      string   `json:"predicate"`
    Object         string   `json:"object"`
    EvidenceIDs    []string `json:"evidence_ids"`
    Strength       string   `json:"strength"`
    Confidence     float64  `json:"confidence"`
    EffectiveDate  string   `json:"effective_date,omitempty"`
    ExpirationDate string   `json:"expiration_date,omitempty"`
    IsCurrent      bool     `json:"is_current"`
    IsDirect       bool     `json:"is_direct"`
}
```

Example:

```json
{
  "candidate_id": "person-123",
  "predicate": "led_project",
  "object": "backend service migration",
  "evidence_ids": ["doc-17", "slack-42"],
  "strength": "strong",
  "confidence": 0.91,
  "effective_date": "2026-03-15",
  "is_current": true,
  "is_direct": true
}
```

The extraction prompt should forbid unsupported interpretation:

```text
Extract only claims directly supported by the supplied evidence.

For each claim:
- Identify the candidate.
- Normalize the predicate.
- Include supporting evidence IDs.
- Distinguish direct statements from inference.
- Capture dates and currentness when available.
- Do not infer expertise solely from job title.
- Do not convert participation into ownership.
- Return no claim when the evidence is insufficient.
```

## Keep scoring transparent

The code should calculate the score from normalized claims:

```go
func scoreCandidate(
    candidate CandidateState,
) CandidateScore {
    score := 0.0

    for _, claim := range candidate.Claims {
        base := claimWeight(claim.Predicate)

        if claim.IsDirect {
            base *= 1.2
        }

        if claim.IsCurrent {
            base *= 1.15
        }

        base *= claim.Confidence
        score += base
    }

    score -= contradictionPenalty(candidate.Claims)
    score -= staleEvidencePenalty(candidate.Claims)

    return CandidateScore{
        CandidateID: candidate.Candidate.ID,
        Score:       score,
    }
}
```

For example:

```go
func claimWeight(predicate string) float64 {
    switch predicate {
    case "explicitly_identified_as_expert":
        return 5.0
    case "owned_system":
        return 4.5
    case "technical_lead":
        return 4.0
    case "authored_design":
        return 3.5
    case "implemented_component":
        return 2.5
    case "participated_in_project":
        return 1.0
    default:
        return 0.0
    }
}
```

The exact weights will need tuning against evaluations. The important point is that the model emits claims, not an unexplained score.

## Use the LLM proposal as a bounded selection step

The proposal call should see only:

* Original query.
* Validated constraints.
* Ranked candidates.
* Compact normalized claims.
* Contradictions and missing evidence.

It should not receive another open-ended tool loop.

```json
{
  "status": "resolved",
  "candidate_id": "person-123",
  "reasoning_summary": [
    "Matches the current Moveworks affiliation constraint",
    "Last name is Wang",
    "Has direct evidence of backend service ownership",
    "Has two recent design and leadership signals"
  ],
  "confidence": "high"
}
```

## Verification should be independent

The verifier should not merely ask, “Does this answer look good?” Give it explicit obligations:

```go
type VerificationResult struct {
    Accepted               bool        `json:"accepted"`
    ConstraintChecks       []Check     `json:"constraint_checks"`
    SupportedClaims        []string    `json:"supported_claims"`
    UnsupportedClaims      []string    `json:"unsupported_claims"`
    Contradictions         []string    `json:"contradictions"`
    StrongerCandidateFound bool        `json:"stronger_candidate_found"`
    Confidence             string      `json:"confidence"`
}
```

The verifier should check:

```text
1. Does the proposed person satisfy every hard constraint?
2. Does each explanatory statement have supporting evidence?
3. Is the evidence about the same person?
4. Is the evidence current enough for the query?
5. Is “expert” justified by substantial evidence rather than a title or mention?
6. Does another candidate have materially stronger evidence?
7. Are there unresolved contradictions?
```

For higher reliability, use a separate model call with no access to the proposal-generation reasoning, only its result and the evidence state.

## How your existing `ToolLoop` fits

You do not necessarily need to remove it. Use it inside tightly scoped stages:

```text
ConstrainedSearchOrchestrator
├── Plan call
├── Candidate generation
├── Hard-filter code
├── Evidence retrieval ToolLoop
│   └── Only evidence-search tools exposed
├── Claim extraction call
├── Ranking code
├── Proposal call
└── Verification call
```

For the evidence loop, the system prompt could say:

```text
You are collecting evidence for one already-identified candidate.

You may only search for evidence relevant to the supplied expertise
dimensions. Do not introduce new candidates. Do not answer the original
user query.

Continue searching until:
- each requested evidence dimension has been attempted,
- pagination is complete or result limits are reached,
- and at least two independent strong sources are found,
  or the available evidence is exhausted.

Return the evidence IDs gathered and identify any dimensions for which
no evidence was found.
```

This is much safer than giving a single autonomous loop all tools and asking it to manage every stage.

## One adjustment to your diagram

I would add an explicit completeness gate before ranking:

```text
User query
   ↓
LLM produces structured constraint plan
   ↓
Code validates and stores plan
   ↓
Candidate-generation tools
   ↓
Code filters hard constraints
   ↓
Evidence retrieval per candidate
   ↓
LLM extracts normalized claims
   ↓
Code checks search completeness
   ↓
Code scores and ranks candidates
   ↓
LLM proposes final candidate
   ↓
Verifier checks constraints and support
   ↓
Answer with evidence and confidence
```

Without that gate, the system may confidently rank an incomplete candidate set. The code should know whether candidate pagination completed, evidence dimensions were attempted, and retrieval budgets were exhausted before allowing a definitive answer.
