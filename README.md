# NOX Writeups

Case studies from **NOX**, a personal, self-hosted multi-agent lab: one human owner, multiple LLM agents from different vendors, a Mac mini as hub, and a working rule of *measurement before claims*.

These are not papers. They are engineering case studies written to the standard we hold our own lab to: preregistered where possible, confounds documented, limits stated, and no green number allowed to authorize more than it proves.

| Writeup | Topic |
|---|---|
| [Onboarding an Untrusted Agent Binary](w1-untrusted-agent-binary.md) | AI control in practice: enforcing — not requesting — constraints on a third-party agent binary (xAI's Grok CLI); four classes of executed false-greens caught by adversarial model-vs-model review |
| [Falsifying Our Own Memory Architecture](w2-memory-arena.md) | A small preregistered eval (n=59) of memory-governance machinery, including a budget confound that inverted the result and a self-bias objection tested with a third-party judge |
| [NOX](front.md) | The shop window: who does what, and the only exhibits that already have a public text |
| [The lab behind the writeups](lab.md) | The rough structure a third party needs: who decides, what the two texts are slices of, and what is deliberately not here |

## Transparency

The lab's owner designed the processes, made every gate decision, and approved publication. The resident LLM agents — who are also the builders, reviewers, and in one case the system under test — drafted and edited these texts. That division of labor is disclosed because it *is* the subject matter: humans deciding gates while models build and red-team each other is the configuration these writeups are about.

No internal hostnames, credentials, or private paths appear in these documents.
