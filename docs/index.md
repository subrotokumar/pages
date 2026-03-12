---
hide:
    - toc
    - navigation
    - footer

---

<style>
.md-grid {
  max-width: 100%;
}
.md-content__inner {
  max-width: 100%;
  padding: 0 3rem;
}
</style>

<img src="https://github.com/subrotokumar/subrotokumar/raw/main/assets/banner.png">
<div style="max-width: 960px; margin: 0 auto; padding: 3.5rem 1rem;">

<h1 style="font-size: 2.75rem; font-weight: 700; margin-bottom: 0.75rem;">Engineering Pages</h1>

<p style="font-size: 1.15rem; color: #6b7280; max-width: 760px;">
  Notes from building and breaking things. Lessons extracted from production—written with context, not theory. Focused on trade-offs, failure modes, and operational reality.
</p>

<hr style="margin: 3.5rem 0;" />

<h2 style="margin-bottom: 0.75rem;">Why this exists</h2>

<p style="max-width: 760px;">
  This is a personal, developer-first documentation space. It exists to capture things that only become obvious after systems misbehave—under load, during incidents, or at scale. These notes are written to preserve reasoning, not just outcomes.
</p>

<p style="max-width: 760px; color: #6b7280;">
  Think of this as an externalized memory for design decisions, sharp edges, and lessons that are expensive to relearn.
</p>

<hr style="margin: 3.5rem 0;" />

<h2>What lives here</h2>

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(260px, 1fr)); gap: 1.25rem; margin-top: 1.75rem;">

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <strong>Production behavior</strong>
    <p style="margin-top: 0.6rem; color: #6b7280;">
      How systems actually behave once they leave localhost—latency, backpressure, partial failures, and surprises.
    </p>
  </div>

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <strong>Design trade-offs</strong>
    <p style="margin-top: 0.6rem; color: #6b7280;">
      The "why" behind decisions, including alternatives that were rejected and the costs that came with each choice.
    </p>
  </div>

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <strong>Failure analysis</strong>
    <p style="margin-top: 0.6rem; color: #6b7280;">
      Incidents, near-misses, and outages—broken down into symptoms, causes, and corrective actions.
    </p>
  </div>

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <strong>Operational notes</strong>
    <p style="margin-top: 0.6rem; color: #6b7280;">
      Runbooks, checks, configs, and patterns that matter when systems are already on fire.
    </p>
  </div>

</div>

<hr style="margin: 3.5rem 0;" />

<h2>Areas of focus</h2>

<ul style="max-width: 760px;">
  <li>Backend systems and service design (Go, Java, Python, TypeScript)</li>
  <li>Distributed systems, consistency, and failure modes</li>
  <li>DevOps, platform tooling, and automation</li>
  <li>Observability, debugging, and incident response</li>
  <li>Performance tuning and scalability limits</li>
</ul>

<hr style="margin: 3.5rem 0;" />

<h2>How to read this</h2>

<p style="max-width: 760px;">
  This is not linear documentation. Jump around. Skim aggressively. Read deeply only when something breaks or feels unfamiliar.
</p>

<p style="max-width: 760px; color: #6b7280;">
  If a note here helps you avoid repeating a mistake—or gives you language to explain one—you’ve extracted the value.
</p>

<hr style="margin: 3.5rem 0;" />

<blockquote style="font-style: italic; color: #374151; max-width: 760px;">
  Reliable systems aren’t built by avoiding failure. They’re built by understanding it.
</blockquote>

<p style="margin-top: 3.5rem; font-size: 0.875rem; color: #9ca3af;">
  Living documentation — updated as systems evolve and assumptions break.
</p>

</div>
