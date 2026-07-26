## Query

```
what is CMDB
```

## Fixed

```
{
  "nodes": [
    {
      "deps": [],
      "id": "q",
      "op": "reformulate_query"
    },
    {
      "args": {
        "corpus": "all"
      },
      "deps": [
        "q"
      ],
      "id": "lex",
      "op": "lexical_retrieval"
    },
    {
      "args": {
        "corpus": "all"
      },
      "deps": [
        "q"
      ],
      "id": "sem",
      "op": "semantic_retrieval"
    },
    {
      "deps": [
        "lex",
        "sem"
      ],
      "id": "m",
      "op": "merge"
    },
    {
      "deps": [
        "m"
      ],
      "id": "f",
      "op": "featurize"
    },
    {
      "args": {
        "k": 50
      },
      "deps": [
        "f"
      ],
      "id": "cap",
      "op": "truncate"
    },
    {
      "args": {
        "method": "bge"
      },
      "deps": [
        "cap"
      ],
      "id": "rr",
      "op": "rerank"
    },
    {
      "deps": [
        "rr"
      ],
      "id": "acl",
      "op": "filter"
    },
    {
      "args": {
        "k": 5
      },
      "deps": [
        "acl"
      ],
      "id": "top",
      "op": "truncate"
    }
  ],
  "output": "top"
}
```


## LLM-Recipe

```
{"nodes":[{"id":"n1","op":"reformulate_query","deps":[],"args":{"org":"default","query":"what is CMDB"}},
          {"id":"n2","op":"lexical_retrieval","deps":["n1"],"args":{"org":"default","query":"n1","corpus":"indexed_data,file_indexed_data,form,kb,faq,file,message,ticket,pull_request,crm_object"}},
          {"id":"n3","op":"semantic_retrieval","deps":["n1"],"args":{"org":"default","query":"n1","corpus":"indexed_data,file_indexed_data,form,kb,faq,file,message,ticket,pull_request,crm_object"}},
          {"id":"n4","op":"merge","deps":["n2","n3"],"args":{}},{"id":"n5","op":"truncate","deps":["n4"],"args":{"k":50}},
          {"id":"n6","op":"rerank","deps":["n5"],"args":{"method":"bge","org":"default","query":"n1"} }],"output":"n6"}
```

## LLM-Code

```starlark
q = reformulate_query(query=query)
lex_hits = lexical_retrieval(query=q, corpus="all", engine="documents", num_results=100)
sem_hits = semantic_retrieval(query=q, corpus="all")
hits = merge(lists=[lex_hits, sem_hits])
hits = featurize(hits=hits, query=q)
hits = truncate(hits=hits, k=50)
hits = rerank(hits=hits, method="bge", query=q)
result = summarize(hits=hits, query=q)
```

