---
name: full-text-search
description: "Full-text search internals and trade-offs: inverted indexes, analysis/tokenization, relevance scoring, fuzzy matching and typo tolerance in Elasticsearch, PostgreSQL full-text search (tsvector, tsquery, trigram typo handling), and when to use which. Use when implementing search features, choosing between Elasticsearch and Postgres search, or tuning relevance and fuzzy matching."
---

# Full-Text Search: A Deep Dive

Full-text search is a sophisticated technique for searching text data that goes far beyond simple pattern matching. Let me explain how systems like Elasticsearch work and how you can achieve similar capabilities in PostgreSQL.

## The Core Concept: Inverted Index

At the heart of full-text search engines lies the **inverted index**, which works opposite to how you might intuitively think about documents:

**Normal structure**: Document → Words it contains
**Inverted index**: Word → Documents containing it

For example, if you have three documents:

- Doc1: "The quick brown fox"
- Doc2: "The lazy dog"
- Doc3: "Quick brown dogs"

The inverted index would look like:

- "quick" → [Doc1, Doc3]
- "brown" → [Doc1, Doc3]
- "fox" → [Doc1]
- "lazy" → [Doc2]
- "dog/dogs" → [Doc2, Doc3]

This structure makes searching incredibly fast because you can instantly look up which documents contain a term, rather than scanning through every document.

## How Elasticsearch Works

Elasticsearch is built on **Apache Lucene**, which provides the core indexing and search functionality. Here's the process:

### 1. **Analysis (Indexing Phase)**

When you index a document, the text goes through an analysis pipeline:

- **Tokenization**: Breaking text into individual terms ("Quick brown fox" → ["Quick", "brown", "fox"])
- **Lowercasing**: Normalizing case ("Quick" → "quick")
- **Stemming**: Reducing words to root form ("running" → "run", "dogs" → "dog")
- **Stop word removal**: Filtering common words like "the", "a", "is"
- **Synonym handling**: Treating related terms similarly

### 2. **Relevance Scoring**

Elasticsearch uses algorithms like **BM25** (Best Matching 25) or **TF-IDF** (Term Frequency-Inverse Document Frequency) to rank results:

- **Term Frequency (TF)**: How often does the term appear in this document?
- **Inverse Document Frequency (IDF)**: How rare is this term across all documents?
- **Field Length Normalization**: Shorter documents get boosted when they contain the term

A term that appears frequently in one document but rarely across the corpus gets a high score.

### 3. **Fuzzy Matching and Typo Tolerance**

Elasticsearch handles typos using several techniques:

**Fuzzy queries** based on **Levenshtein distance** (edit distance):

- "quik" can match "quick" (1 edit: insert 'c')
- You can specify maximum edit distance (typically 1 or 2)

**N-grams**: Breaking words into overlapping chunks:

- "search" → ["se", "ea", "ar", "rc", "ch"]
- Even with typos, many n-grams still match

**Phonetic matching**: Algorithms like Soundex or Metaphone match words that sound similar.

## PostgreSQL Full-Text Search

PostgreSQL has built-in full-text search capabilities that are surprisingly powerful, though not as feature-rich as Elasticsearch.

### Key Components:

**1. tsvector (Text Search Vector)**
This is PostgreSQL's version of an inverted index representation:

sql

`SELECT to_tsvector('english', 'The quick brown fox jumps');
-- Result: 'brown':3 'fox':4 'jump':5 'quick':2`

Notice it removed "the", stemmed "jumps" to "jump", and stored positions.

**2. tsquery (Text Search Query)**
Represents what you're searching for:

sql

`SELECT to_tsquery('english', 'quick & fox');
-- Searches for documents with both "quick" AND "fox"`

**3. GIN or GiST Indexes**
These specialized index types make searching tsvector columns fast:

sql

`CREATE INDEX idx_fts ON articles 
USING GIN (to_tsvector('english', content));`

### Example Usage:

sql

- `- Create a table with full-text searchCREATE TABLE articles ( id SERIAL PRIMARY KEY, title TEXT, content TEXT, search_vector tsvector
);- Populate the search vectorUPDATE articles
SET search_vector = to_tsvector('english', coalesce(title, '') || ' ' || coalesce(content, ''));- Create indexCREATE INDEX ON articles USING GIN(search_vector);- Search with rankingSELECT title, ts_rank(search_vector, query) AS rank
FROM articles, to_tsquery('english', 'postgresql & search') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;`

### Typo Handling in PostgreSQL:

PostgreSQL doesn't have built-in fuzzy matching like Elasticsearch, but you can use:

**1. pg_trgm extension** (trigram matching):

sql

`CREATE EXTENSION pg_trgm;

SELECT * FROM articles 
WHERE content % 'postgrsql';  -- Will match "postgresql"`

**2. Similarity scoring**:

sql

`SELECT word, similarity(word, 'postgrsql') AS sim
FROM words
WHERE word % 'postgrsql'
ORDER BY sim DESC;`

## When to Use What?

**Use PostgreSQL FTS when:**

- You're already using PostgreSQL
- Your search needs are moderate
- You want to avoid managing another system
- You need transactional consistency with your data

**Use Elasticsearch when:**

- You need advanced relevance tuning
- You have massive scale (billions of documents)
- You need sophisticated typo handling
- You want features like highlighting, suggestions, and faceting
- You're building a search-first application
