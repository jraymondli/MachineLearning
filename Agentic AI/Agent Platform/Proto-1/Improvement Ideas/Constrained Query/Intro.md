For constrained queries, the biggest improvement is to stop treating search as a single retrieval step. Make the agent run a small **constraint-solving workflow**:

```text
Parse constraints
      ↓
Generate candidates
      ↓
Gather evidence per candidate
      ↓
Apply hard filters
      ↓
Rank remaining candidates
      ↓
Verify the final answer
```

For your example:

> “Who is a Moveworks backend systems expert with last name Wang?”

the constraints are not equal:

```json
{
  "hard_constraints": {
    "company": "Moveworks",
    "last_name": "Wang"
  },
  "soft_constraints": {
    "expertise": "backend systems"
  }
}
```

`Moveworks` and `Wang` should normally be exact filters. “Backend systems expert” requires semantic interpretation and evidence.

## 1. Add a structured query-planning step

Before tool use, have the model produce a search plan:

```json
{
  "intent": "find_person",
  "target_count": 1,
  "constraints": [
    {
      "field": "company",
      "value": "Moveworks",
      "type": "hard"
    },
    {
      "field": "last_name",
      "value": "Wang",
      "type": "hard"
    },
    {
      "field": "expertise",
      "value": "backend systems",
      "type": "soft",
      "expansions": [
        "distributed systems",
        "backend services",
        "platform engineering",
        "infrastructure",
        "service architecture",
        "scalability",
        "APIs",
        "databases"
      ]
    }
  ],
  "strategy": [
    "find people matching hard identity constraints",
    "collect expertise evidence for each candidate",
    "rank candidates",
    "verify strongest candidate"
  ]
}
```

Do not rely only on the LLM to remember this plan in prose. Store it in code and pass the current state back during each turn.

## 2. Separate candidate generation from evidence retrieval

A weak agent sends one query like:

```text
Moveworks backend expert Wang
```

That often returns the most semantically similar document, not the correct person.

Use two phases instead.

### Candidate generation

Search for people satisfying objective constraints:

```text
company = Moveworks
last_name = Wang
```

This should return a candidate set:

```json
[
  {
    "person_id": "p123",
    "name": "Alice Wang",
    "company": "Moveworks",
    "role": "Staff Software Engineer"
  },
  {
    "person_id": "p456",
    "name": "Bob Wang",
    "company": "Moveworks",
    "role": "Engineering Manager"
  }
]
```

### Evidence retrieval

For each candidate, search separately:

```text
Alice Wang backend distributed systems infrastructure
Bob Wang backend distributed systems infrastructure
```

This prevents one highly visible candidate from dominating the entire retrieval process.

## 3. Enforce hard constraints in code

Do not let the LLM decide whether a hard constraint is “close enough.”

```go
func satisfiesHardConstraints(
    candidate Candidate,
    plan QueryPlan,
) bool {
    if !strings.EqualFold(candidate.LastName, plan.LastName) {
        return false
    }

    if !strings.EqualFold(candidate.Company, plan.Company) {
        return false
    }

    return true
}
```

Semantic similarity should not override:

* Wrong last name
* Wrong company
* Former employee when the query implies current employment
* Different person with a similar name

Use the LLM mainly for ambiguous constraints such as expertise, ownership, influence, or relevance.

## 4. Track candidates explicitly

Maintain a candidate table outside the model:

```go
type CandidateState struct {
    PersonID        string
    Name            string
    HardMatch       bool
    Evidence        []Evidence
    ExpertiseScore  float64
    RecencyScore    float64
    Confidence      float64
    RejectedReasons []string
}
```

After every tool call, update this state. Then present a compact representation to the LLM:

```text
Candidate 1: Alice Wang
- Hard constraints: passed
- Evidence:
  - Owned backend service migration
  - Authored distributed systems design
- Evidence dates: 2025–2026
- Current confidence: 0.82

Candidate 2: Bob Wang
- Hard constraints: passed
- Evidence:
  - Managed platform team
  - No direct backend ownership found
- Current confidence: 0.51
```

This is much more reliable than asking the model to reconstruct candidate state from a long message history.

## 5. Distinguish expertise evidence from mere mentions

Not every document mentioning “backend” establishes expertise.

Define evidence strength:

```text
Strong evidence
- Explicitly described as owner, expert, architect, or technical lead
- Authored a relevant design document
- Led a backend migration or infrastructure project
- Repeated code ownership in relevant services

Medium evidence
- Repeated participation in backend projects
- Relevant job title and project contributions
- Frequently consulted on related technical issues

Weak evidence
- Appears in a meeting
- Mentioned in one backend discussion
- Has “software engineer” in title
- Search result contains related keywords
```

You can encode this as a rubric:

```json
{
  "expertise_scoring": {
    "explicit_owner": 5,
    "design_author": 4,
    "technical_lead": 4,
    "repeated_contributor": 3,
    "relevant_role": 2,
    "single_mention": 1
  }
}
```

The agent should produce a score from evidence, not from intuition alone.

## 6. Add temporal reasoning

Constrained search often fails because the agent finds correct but outdated evidence.

Track:

* Current versus former employee
* Recent versus old projects
* Current team
* Superseded ownership
* Historical versus active expertise

For example:

```json
{
  "person": "Alice Wang",
  "evidence": "Owned backend platform",
  "valid_from": "2023-04-01",
  "valid_to": "2024-10-01",
  "current": false
}
```

A newer source saying someone else took ownership should override older evidence.

Your system instructions should include:

```text
Prefer current evidence. Do not infer current ownership from historical
documents when newer evidence contradicts or supersedes it.
```

## 7. Improve tool design

The quality of the agent is often limited more by tool APIs than by the system prompt.

Prefer focused tools such as:

```text
search_people(filters)
get_person_profile(person_id)
search_person_evidence(person_id, concepts)
get_recent_projects(person_id)
search_documents(query, filters)
search_code_ownership(person_id, repositories)
```

Avoid exposing only a generic:

```text
search_everything(query)
```

A generic search tool encourages the model to issue large, vague queries.

A good people-search input might be:

```json
{
  "company": "Moveworks",
  "last_name": "Wang",
  "employment_status": "current",
  "limit": 50
}
```

Then evidence search:

```json
{
  "person_id": "p123",
  "concepts": [
    "backend systems",
    "distributed systems",
    "platform infrastructure"
  ],
  "date_range": {
    "from": "2024-01-01"
  },
  "limit": 20
}
```

## 8. Avoid truncating critical results

Your 6,000-character tool-result limit can silently remove important candidates.

Instead of returning raw text, return:

```json
{
  "items": [
    {
      "id": "p123",
      "name": "Alice Wang",
      "summary": "Staff engineer on platform backend",
      "score": 0.86
    }
  ],
  "total": 27,
  "has_more": true,
  "next_cursor": "cursor-2"
}
```

Then fetch details by ID.

Also tell the model explicitly:

```text
When has_more is true and the current evidence is insufficient, paginate
before concluding.
```

Otherwise the agent may assume the first page is complete.

## 9. Add a verifier step

Do not let the same reasoning pass both select and approve the answer without challenge.

After the agent chooses a candidate, run a verification prompt:

```text
Verify whether the proposed answer satisfies every query constraint.

Query:
Who is a Moveworks backend systems expert with last name Wang?

Proposed candidate:
Alice Wang

Evidence:
...

Return:
- hard_constraints_passed
- supporting_evidence
- contradictory_evidence
- unsupported_assumptions
- confidence
- final_decision
```

Example output:

```json
{
  "hard_constraints_passed": true,
  "supporting_evidence": [
    "Current Moveworks employee",
    "Last name is Wang",
    "Led backend service architecture work"
  ],
  "contradictory_evidence": [],
  "unsupported_assumptions": [
    "No source explicitly uses the word expert"
  ],
  "confidence": "medium",
  "final_decision": "accept"
}
```

This catches a surprising number of premature answers.

## 10. Set stopping criteria

Your current loop stops when the LLM emits no tool calls. That lets the model stop too early.

Add application-level completion rules:

```go
func searchComplete(state SearchState) bool {
    return state.CandidateSearchCompleted &&
        state.HardConstraintsVerified &&
        state.TopCandidateEvidenceCount >= 2 &&
        state.VerificationCompleted
}
```

If those conditions are not met, remind the model:

```text
The search is incomplete because candidate enumeration has not finished.
Continue using tools.
```

A reasonable policy might require:

* Candidate search completed
* At least two independent supporting pieces of evidence
* Hard constraints verified
* No unresolved stronger candidate
* Final verification completed

## 11. Use explicit uncertainty states

Avoid forcing every query into a single definitive answer.

Your result schema can support:

```json
{
  "status": "resolved | ambiguous | insufficient_evidence | no_match",
  "best_candidate": {},
  "alternatives": [],
  "confidence": "high | medium | low",
  "missing_evidence": []
}
```

For example:

```json
{
  "status": "ambiguous",
  "best_candidate": {
    "name": "Alice Wang"
  },
  "alternatives": [
    {
      "name": "Bob Wang"
    }
  ],
  "confidence": "medium",
  "missing_evidence": [
    "No authoritative source explicitly identifies the primary backend expert"
  ]
}
```

That is better than hallucinating certainty.

## 12. Build constrained-query evaluations

Create a benchmark that tests more than answer correctness.

Include query types such as:

```text
Exact identity
- Find the Moveworks engineer with last name Wang

Multiple constraints
- Find a backend expert at Moveworks with last name Wang

Temporal
- Who currently owns the forms indexing pipeline?

Negation
- Find a Wang who works on backend systems but not search relevance

Superlative
- Who has the most experience with OpenSearch migrations?

Multi-hop
- Who worked on the migration and later reviewed the reliability design?
```

Measure:

* Hard-constraint satisfaction
* Candidate recall
* Evidence precision
* Citation correctness
* Currentness
* Unsupported-claim rate
* Tool-call count
* Latency
* Token and LLM cost

The critical metric is not only whether the correct name appeared. It is whether the answer was supported and all constraints were satisfied.

## Recommended architecture

For your existing tool loop, I would evolve it toward:

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
Code scores and ranks candidates
   ↓
LLM proposes final candidate
   ↓
Verifier checks constraints and support
   ↓
Answer with evidence and confidence
```

The core principle is:

> Use retrieval to generate evidence, code to enforce objective constraints, and the LLM to interpret ambiguous evidence.

A stronger system prompt helps, but the largest gains will come from candidate state, structured tool APIs, hard-filter enforcement, and a separate verification stage.
