---
hide:
    - toc
---

# Kubernetes Internals and Platform Operations

Production deep dives into Kubernetes control plane mechanics, pod scheduling bottlenecks, autoscaling physics, and zero trust configuration management.

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 1.25rem; margin-top: 1.75rem;">

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <a href="why_hpa_lags_behind_traffic_spikes_by_design/" style="text-decoration: none; font-weight: 700; font-size: 1.1rem; color: inherit;">Why HPA Lags Behind Traffic Surges</a>
    <p style="margin-top: 0.6rem; color: #6b7280; font-size: 0.95rem;">
      Analyzing the cumulative delay across cAdvisor scraping, metrics-server polling, and controller reconciliation loops. Why reactive CPU scaling fails and how KEDA changes causality.
    </p>
  </div>

  <div style="border: 1px solid #e5e7eb; border-radius: 10px; padding: 1.25rem;">
    <a href="managing_secrets_on_kubernetes_with_sops_and_age/" style="text-decoration: none; font-weight: 700; font-size: 1.1rem; color: inherit;">Managing Secrets with SOPS and Age</a>
    <p style="margin-top: 0.6rem; color: #6b7280; font-size: 0.95rem;">
      A GitOps architecture for committing encrypted secrets into version control. Envelope encryption mechanics, in-memory KSOPS decryption in ArgoCD, and compliance audit boundaries.
    </p>
  </div>

</div>