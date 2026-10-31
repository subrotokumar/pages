---
hide:
    - toc
    - footer
---

# System Design and Distributed Systems

Concrete engineering notes on designing infrastructure that survives beyond the whiteboard. Focuses on distributed coordination, storage engines, network socket performance, caching failures, and probabilistic algorithms.

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 1.25rem; margin-top: 1.75rem;">

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <a href="caching/cache_stampede/" style="text-decoration: none; font-weight: 700; font-size: 1.1rem; color: inherit;">Cache Stampede</a>
    <p style="margin-top: 0.6rem; color: #6b7280; font-size: 0.95rem;">
      Distributed locking hazards, convoy effects, probabilistic early expiration mathematics, and avoiding secondary database failures during post-outage cluster cutover.
    </p>
  </div>

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <a href="deep_dive/networking/" style="text-decoration: none; font-weight: 700; font-size: 1.1rem; color: inherit;">Network Protocols Reality</a>
    <p style="margin-top: 0.6rem; color: #6b7280; font-size: 0.95rem;">
      Packet switching physics, kernel socket queues, TCP congestion control algorithms (Cubic vs BBR), head of line blocking in HTTP/2, and mitigating cascading retry storms.
    </p>
  </div>

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <a href="deep_dive/data_structure_for_big_data/" style="text-decoration: none; font-weight: 700; font-size: 1.1rem; color: inherit;">Probabilistic Data Structures</a>
    <p style="margin-top: 0.6rem; color: #6b7280; font-size: 0.95rem;">
      Trading exactness for orders of magnitude in space and throughput: Bloom filters with Kirsch-Mitzenmacher hashing, Count-Min sketches with conservative updates, and HyperLogLog.
    </p>
  </div>

</div>
