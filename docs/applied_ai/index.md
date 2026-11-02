---
hide:
    - toc
    - footer
---

# Applied AI Architecture

Practical engineering notes on deploying artificial intelligence systems into production infrastructure. Focuses on retrieval augmented generation, bi-encoder and cross-encoder pipelines, token economics, latency bounds, and offline evaluation rigor.

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 1.25rem; margin-top: 1.75rem;">

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <a href="rag/" style="text-decoration: none; font-weight: 700; font-size: 1.1rem; color: inherit;">Production RAG Architecture</a>
    <p style="margin-top: 0.6rem; color: #6b7280; font-size: 0.95rem;">
      Moving beyond naive vector search: hybrid BM25 and dense retrieval, reciprocal rank fusion, cross-encoder rerankers, parent document chunking, and context window economics.
    </p>
  </div>

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <a href="token_cost_of_large_tool_sets/" style="text-decoration: none; font-weight: 700; font-size: 1.1rem; color: inherit;">Token Cost of Large Tool Sets</a>
    <p style="margin-top: 0.6rem; color: #6b7280; font-size: 0.95rem;">
      How to cut tool-definition overhead: deferred loading, prompt caching, bounded results, and real-world lessons.
    </p>
  </div>

</div>