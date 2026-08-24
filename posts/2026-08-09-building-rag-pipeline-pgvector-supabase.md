---
title: Building a RAG pipeline with pgvector and Supabase
date: 2026-08-09T10:00:00.000Z
author: Jyothish Narayan
category: AI
summary: A practical walkthrough of setting up semantic search for your applications using pgvector and Supabase — open-source tools that handle vector embeddings without the complexity of dedicated vector databases.
image: images/thumb-rag-pipeline.png.png
---

Retrieval-Augmented Generation (RAG) is one of the most practical AI patterns you can add to any application. Instead of relying solely on an LLM's training data, RAG lets your app pull relevant context from your own data before generating a response.

## What we're building

A simple semantic search pipeline that:
- Accepts a text query from the user
- Converts it to a vector embedding
- Finds the most semantically similar documents in Supabase
- Returns the top matches for use in an LLM prompt

## Why pgvector + Supabase

pgvector is a PostgreSQL extension that adds vector storage and similarity search directly to your existing database. Supabase ships with pgvector enabled by default, which means you get vector search without spinning up a separate service like Pinecone or Weaviate.

## Setting up the database

First, enable pgvector in your Supabase project and create a table to store your documents and their embeddings.

```sql
-- Enable pgvector
create extension if not exists vector;

-- Create documents table
create table documents (
  id bigserial primary key,
  content text,
  embedding vector(1536)
);

-- Create an index for fast similarity search
create index on documents 
using ivfflat (embedding vector_cosine_ops)
with (lists = 100);
```

## Generating embeddings

Use OpenAI's embedding model to convert text to vectors, then store them in Supabase.

```javascript
import { createClient } from '@supabase/supabase-js';
import OpenAI from 'openai';

const supabase = createClient(process.env.SUPABASE_URL, process.env.SUPABASE_KEY);
const openai = new OpenAI();

async function embedAndStore(text) {
  const { data } = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: text,
  });
  
  await supabase.from('documents').insert({
    content: text,
    embedding: data[0].embedding
  });
}
```

## Querying with semantic similarity

```javascript
async function semanticSearch(query, limit = 5) {
  const { data: embedding } = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: query,
  });

  const { data } = await supabase.rpc('match_documents', {
    query_embedding: embedding[0].embedding,
    match_threshold: 0.7,
    match_count: limit
  });

  return data;
}
```

## Wrapping up

This setup gives you a production-ready semantic search pipeline with no additional infrastructure. Your vector data lives alongside your regular application data in Supabase, making it easier to manage relationships, permissions, and backups.

In a follow-up post we'll look at combining this with a streaming LLM response to build a full RAG chatbot.
