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


