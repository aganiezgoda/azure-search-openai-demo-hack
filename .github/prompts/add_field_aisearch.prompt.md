This RAG app works well.

When preprocessing the source documents - creating their embeddings and throwing into Azure AI Search - I would like to add additional step. 

Each document should have a priority on a scale 1-3. To implement it add an additional field in the Azure AI Search index called "priority". When adding chunks from a document to the index assign them a priority on the scale 1-3 - the number should be random for now. All chunks from one document should have the same priority.