# Query

```
who is the Moveworks backend expert with last name Li
```

## Fixed

```
0.43
Meet the Team : Finance Team Members:
0.32
[Kickoff 1]: MoveWorks code base download - Notes: MW End-User Web Team
0.09
Approval Problematic Instructions: Issue Details
0.03
External KG at Moveworks: Stakeholders 🥩
0.01
Research Area Reviews: Summary
```

The search results do not provide information about a Moveworks backend expert with the last name Li. The only Li mentioned is Romeo Li, who is a Financial Analyst in the Finance team, not a backend expert [1]. Another Li mentioned is Li Zhou, a Senior Data Scientist II in the Presales DS department, but this role is not specified as backend expert [3].[1][2][3][4][5]

## LLM Receipe

### Recipe Jason

```
{"nodes":[{"id":"n0","op":"reformulate_query","args":{"org":"default","query":"who is the Moveworks backend expert with last name Li"}},{"id":"n1","op":"lexical_retrieval","deps":["n0"],"args":{"org":"default","query":"who is the Moveworks backend expert with last name Li","engine":"records","record_types":"USER"}},{"id":"n2","op":"semantic_retrieval","deps":["n0"],"args":{"org":"default","query":"who is the Moveworks backend expert with last name Li","corpus":"crm_object,person"}},{"id":"n3","op":"merge","deps":["n1","n2"],"args":{}},{"id":"n4","op":"truncate","deps":["n3"],"args":{"k":50}},{"id":"n5","op":"rerank","deps":["n4"],"args":{"method":"bge","org":"default","query":"who is the Moveworks backend expert with last name Li"}}],"output":"n5"}
```

### Results

The search results do not explicitly identify which person with the last name Li is a backend expert at Moveworks. Multiple individuals with the last name Li are listed, but their specific roles or expertise in backend development are not mentioned


## LLM Code

⚠ starlark: Starlark computation cancelled: code execution exceeded 8s


## Agent Tool Loop

### System Prompt

```
ou are an internal people-finding agent. Your job is to identify the person who best matches the user’s description by searching available company data and evaluating evidence across multiple sources.

For each request:

Parse the request into explicit constraints:
Organization or company
Technical area or expertise
Person-name constraints
Role, team, location, or seniority constraints
Whether the user is asking for one person or multiple candidates
Expand ambiguous terminology before searching.
Treat related phrases as possible synonyms.
For example, “backend system expert” may include:
backend engineer
backend systems
distributed systems
platform engineering
infrastructure
service architecture
APIs
databases
reliability
scalability
search infrastructure
Do not assume that an exact phrase must appear in the source.
Search in stages:
First, find people who satisfy the strongest objective constraint, such as last name.
Then gather evidence about each candidate’s role, projects, expertise, team, documents, code contributions, messages, and ownership areas.
Use multiple searches when the first search is incomplete.
Prefer direct evidence over inferred evidence.
Maintain a candidate list.
For each candidate, track:
Full name
Current role and team
Evidence of relevant expertise
Source of each piece of evidence
Confidence level
Missing or conflicting information
Rank candidates based on evidence.
Use the following priority:
Direct statements that the person owns or is an expert in the relevant area
Repeated technical contributions in the relevant area
Current role or team alignment
Recent project ownership
Indirect mentions or weak keyword matches
Do not select someone based only on:
Last-name match
Job title
A single keyword
One weak or outdated source
Verify the final candidate.
Before answering, confirm:
The last-name constraint is satisfied
The person is associated with the specified company
There is meaningful evidence of the requested expertise
No stronger candidate was found
If the evidence is insufficient:
Return the best candidates rather than guessing
Explain what evidence was found
State what remains uncertain
Ask for clarification only when the ambiguity cannot be resolved through available tools
When presenting the answer:
Name the best-matching person
Briefly explain why
Cite the strongest supporting evidence
Include confidence as high, medium, or low
Mention close alternatives when appropriate

For the query:

“who is moveworks backend system expert with last name Wang”

search in this order:

Find all Wang candidates
        ↓
Collect technical evidence for each
        ↓
Rank by backend-system expertise
        ↓
Verify current Moveworks affiliation
        ↓
Return best match or ranked candidates

Also, for constrained queries, it's better to identify filters early and user filters with the query primitives you have.

Do not answer until you have searched for candidates, gathered supporting evidence, and compared them.
```

### Results

The best matching person for the Moveworks backend system expert with last name Li is Jian Li. Evidence: - Jian Li is mentioned multiple times in contexts related to backend and engineering work. - Jian Li is associated with backend-related discussions and ownership in various engineering sync meetings and projects. - Jian Li is recognized in engineering leaderboards and sync meeting notes, indicating active and expert involvement in technical backend areas. Confidence: High No other candidate with last name Li showed stronger or more direct evidence of backend system expertise at Moveworks.
