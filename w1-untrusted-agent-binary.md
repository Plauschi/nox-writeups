# Onboarding an Untrusted Agent Binary

*A case study in enforcing — rather than requesting — constraints on an AI agent. From NOX, a personal multi-agent lab. Written collaboratively: the lab's owner designed the process and made every gate decision; the two resident LLM agents (who were also the builder and the adversarial reviewer in the story below) drafted and edited this text. That division of labor is part of the point.*

## Setting

NOX is a self-hosted multi-agent system on a Mac mini: a human owner and two LLM harnesses (Claude Code and a second, independently-run agent) coordinating over a self-hosted relay, with a shared knowledge vault and strict single-writer rules. We wanted to add a third agent: a vendor-distributed CLI coding agent — xAI's Grok CLI — to run as a restricted worker.

The question that turned this from an install into a research exercise: **what actually binds an agent — and what only appears to?**

## What we found before running anything

We adopted a standing rule early: *capability archaeology before adoption* — measure what the artifact and its ecosystem actually do, source-first, before executing anything.

- **No provenance.** The vendor published neither signatures nor checksums at the time of onboarding (August 2026 — 404 on every conventional location; no Sigstore attestations). Its own `install.sh` verifies nothing — it "verifies" the download by *executing the fresh binary* (`--version`).
- **Two-front download comparison has a sharp limit.** We fetched the binary via the vendor's domain and via its storage backend. Identical object metadata showed both fronts serve *the same stored object* — so the comparison can only detect transit divergence, not a compromised origin. We recorded that as a limit instead of claiming more.
- **Static closure.** The binary is statically linked: the full execution closure is exactly one file, which makes a read-only mount trivial. We established this with three independent static tools and deliberately never used `ldd` — `ldd` resolves dependencies by *invoking the loader*, i.e. it executes parts of the artifact you are trying not to run.
- **The approval layer was not a control surface.** The desktop integration shipping this agent auto-approves every permission request. The agent's own permission engine reads the *host user's* rules file — our instance inherited the owner's entire, unrelated permission ruleset. The effective isolation lever turned out to be a separate `HOME`, which emptied that inheritance; the advertised isolation flags did not.
- **Ambient credentials reach further than expected.** A review round caught that the first execution path would have passed the shell's full environment — including real API keys for unrelated services — into the untrusted binary. First executions now run under `env -i` with an explicit allowlist, and raw child output never enters receipts (digests only).

## Enforcement, not prose

Our standing lesson from earlier incidents: **restrictions must be technically enforced — config, permissions, kernel sandbox — never only written into a system prompt or charter.** Models read "your tool permissions are stronger than any text" as a grant of authority; we had to rewrite exactly that sentence in our own onboarding brief after review.

The final architecture for first execution:

1. **MicroVM with host-enforced egress** (traffic can only leave through a host-side proxy), deny-all network policy for the bootstrap run, including the binary's embedded telemetry endpoint.
2. **A phased protocol** (provenance → artifact + closure → candidate lane → gate harness → bootstrap probe), where every phase needs an owner-signed GO *visible in the shared channel before the first side effect* — a builder's claim that approval happened is not evidence.
3. **One bounded first run**: a one-shot execution grant, issued by the gate layer and *consumed* atomically. The need was measured, not assumed — a review round showed **18 of 40** parallel attempts slipping through the original gate; the fix is an atomic `rename(2)` claim designed to admit exactly one. Argv is rendered from a whitelist rather than validated, and receipts bind attempt-id, grant digest, child-spawn count and a verified sandbox stop.

## What adversarial review actually caught

The builder (one model) and a sealed reviewer (a different model, run independently) iterated in strict rounds. The attestation was granted only in the **fourth** review round; the receipts of rounds one through three record the blockers. Nearly every blocker was the same class in new clothing: **a verdict computed from what the builder reported instead of from raw observations.** Examples that were real, executed false-greens, not theory:

- a "comprehension pass" produced by `all([])` being `True` on an empty check list;
- a preflight that proved a read-only boundary *on one sandbox* while the green verdict pointed at another;
- a guest-side self-report whose PASS constants were reproducible byte-identically without any sandbox;
- a launcher that checked "the sandbox name appears in argv" — satisfied by `--label`, while `--name` targeted a different sandbox.

Three durable principles came out of this, each paid for with a concrete failure:

- **Construct, don't validate.** If a validator must anticipate every bypass, the check has the wrong shape. Render the privileged invocation from declared parameters; refuse everything free-form.
- **Fail closed on absence.** Missing data, empty sets, unparseable output and non-zero exit codes are failures, never passes. A monitor that proves liveness does not prove effect — our single most common failure mode across the whole lab ("status ok, no effect").
- **Exit code 0 is not a release signal.** Our preflight-only path exits 0 and starts nothing; automation must read the receipt's verdict field.

The attestation that finally held is scoped to exact commit, sandbox identity, and rendered argv — and the write-up of its *residual* limits (a millisecond TOCTOU window before `exec`, no long-duration soak, a same-privilege local attacker being out of scope) is part of the artifact, because an attestation that hides its limits is another false-green.

## Why we think this is relevant

This is AI-control engineering at hobby scale: an untrusted model artifact, a control layer built and red-teamed *by other models*, and a human owner whose only scalable contribution is deciding gates — which is precisely the configuration frontier labs describe. The recurring failure class we kept finding in our own code (trusting builder-supplied summaries over raw evidence) is, we suspect, not a hobby-scale problem.

*Residual honesty note: within this delivery lane, the candidate binary was executed exactly once, under the regime above (ledger: executed, marker 1, rc 0). A separate instance of the same vendor CLI later joined the lab as a partner agent — through its own onboarding, with its own isolation profile; that is a different story with its own receipts. The delivery lane itself deliberately stalls at every gate the owner hasn't opened.*
