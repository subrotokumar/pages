---
hide:
    - toc
    - footer
---

# Systems Deep Dives

Low level exploration of physical constraints, kernel metrics, protocol implementations, and probabilistic algorithms that power large scale distributed backends.

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 1.25rem; margin-top: 1.75rem;">

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <a href="networking/" style="text-decoration: none; font-weight: 700; font-size: 1.1rem; color: inherit;">Network Protocols Reality</a>
    <p style="margin-top: 0.6rem; color: #6b7280; font-size: 0.95rem;">
      Packet switching physics, kernel socket queues, TCP congestion control algorithms (Cubic vs BBR), head of line blocking in HTTP/2, and mitigating cascading retry storms.
    </p>
  </div>

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <a href="data_structure_for_big_data/" style="text-decoration: none; font-weight: 700; font-size: 1.1rem; color: inherit;">Probabilistic Data Structures</a>
    <p style="margin-top: 0.6rem; color: #6b7280; font-size: 0.95rem;">
      Trading exactness for orders of magnitude in space and throughput: Bloom filters with Kirsch-Mitzenmacher hashing, Count-Min sketches with conservative updates, and HyperLogLog.
    </p>
  </div>

</div>