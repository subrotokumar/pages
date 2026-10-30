---
hide:
    - toc
    - navigation
    - footer
---

<img src="https://github.com/subrotokumar/subrotokumar/raw/main/assets/banner.png" alt="Engineering Banner">
<div style="max-width: 960px; margin: 0 auto; padding: 3.5rem 1rem;">

<h1 style="font-size: 2.75rem; font-weight: 700; margin-bottom: 0.75rem;">Engineering Pages</h1>

<p style="font-size: 1.15rem; color: #6b7280; max-width: 760px;">
Field notes from building, scaling, and occasionally rescuing distributed infrastructure. Concrete operational realities, hardware limits, architectural compromises, and postmortem findings. No marketing fluff.
</p>

<hr style="margin: 3.5rem 0;" />

<h2 style="margin-bottom: 0.75rem;">Why this exists</h2>

<p style="max-width: 760px;">
Most architectural documentation describes how software works when traffic is gentle and network sockets never stall. These notes focus on what happens after services leave localhost. When connection pools exhaust, kernel packet buffers fill, or autoscalers lag behind sudden traffic surges, theoretical purity disappears.
</p>

<p style="max-width: 760px; color: #6b7280;">
This repository functions as an externalized memory bank for edge cases, non obvious kernel behaviors, and production decisions that are expensive to relearn during outages.
</p>

<hr style="margin: 3.5rem 0;" />

<h2>What lives here</h2>

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(260px, 1fr)); gap: 1.25rem; margin-top: 1.75rem;">

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <a href="system_design/" style="text-decoration: none; font-weight: 700; font-size: 1.1rem; color: inherit;">Production behavior</a>
    <p style="margin-top: 0.6rem; color: #6b7280; font-size: 0.95rem;">
      How systems actually behave once they leave localhost - latency, backpressure, partial failures, and surprises.
    </p>
  </div>

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <a href="kubernetes/" style="text-decoration: none; font-weight: 700; font-size: 1.1rem; color: inherit;">Design trade-offs</a>
    <p style="margin-top: 0.6rem; color: #6b7280; font-size: 0.95rem;">
      The "why" behind decisions, including alternatives that were rejected and the costs that came with each choice.
    </p>
  </div>

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <a href="applied_ai/" style="text-decoration: none; font-weight: 700; font-size: 1.1rem; color: inherit;">Failure analysis</a>
    <p style="margin-top: 0.6rem; color: #6b7280; font-size: 0.95rem;">
      Incidents, near-misses, and outages-broken down into symptoms, causes, and corrective actions.
    </p>
  </div>

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <a href="system_design/deep_dive/networking/" style="text-decoration: none; font-weight: 700; font-size: 1.1rem; color: inherit;">Operational notes</a>
    <p style="margin-top: 0.6rem; color: #6b7280; font-size: 0.95rem;">
      Runbooks, checks, configs, and patterns that matter when systems are already on fire.
    </p>
  </div>

</div>

<hr style="margin: 3.5rem 0;" />

<h2>Reading Philosophy</h2>

<p style="max-width: 760px;">
Treat this documentation as a reference index rather than a textbook. Jump directly to the failure mode you are currently debugging or planning around. If a single page saves you a postmortem or prevents a database lockup during peak traffic, it served its purpose.
</p>

<blockquote style="font-style: italic; color: #374151; max-width: 760px; margin-top: 2rem;">
Reliable systems are rarely built by avoiding failure. They are built by understanding exactly how components break under pressure and ensuring degradation is graceful.
</blockquote>

<p style="margin-top: 3.5rem; font-size: 0.875rem; color: #9ca3af;">
Living engineering repository. Continuously updated as distributed systems evolve and assumptions get tested against reality.
</p>

</div>
