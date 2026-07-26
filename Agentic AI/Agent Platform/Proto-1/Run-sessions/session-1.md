## Fixed

## LLM-Recipe

```
{"nodes":[{"id":"n1","op":"reformulate_query","deps":[],"args":{"org":"default","query":"what is CMDB"}},
          {"id":"n2","op":"lexical_retrieval","deps":["n1"],"args":{"org":"default","query":"n1","corpus":"indexed_data,file_indexed_data,form,kb,faq,file,message,ticket,pull_request,crm_object"}},
          {"id":"n3","op":"semantic_retrieval","deps":["n1"],"args":{"org":"default","query":"n1","corpus":"indexed_data,file_indexed_data,form,kb,faq,file,message,ticket,pull_request,crm_object"}},
          {"id":"n4","op":"merge","deps":["n2","n3"],"args":{}},{"id":"n5","op":"truncate","deps":["n4"],"args":{"k":50}},
          {"id":"n6","op":"rerank","deps":["n5"],"args":{"method":"bge","org":"default","query":"n1"} }],"output":"n6"}
```
