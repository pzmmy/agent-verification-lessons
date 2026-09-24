# When your agents lie to you

**Acceptance lessons from a 4-machine, 3-runtime coding pipeline** (one node since retired).

A field write-up about a problem that shows up the moment you stop running an agent on your own
machine: you can no longer see whether the work actually happened. The pipeline described here ran
LLM coding steps across heterogeneous hosts — a WSL host, two Linux servers and a Windows box,
three different agent runtimes — where **the agent that wrote the code and the party verifying it
were never on the same machine.**

The write-up collects **nineteen ways the verification itself lies** (symptom → cost → the test that
catches it → the fix), three extra constraints that appear only when you cross machine boundaries,
the five-layer architecture that eventually held, and a migration checklist for anyone building the
same thing. Roughly 6,265 words; the whole file is `when-your-agents-lie-to-you.md`.

## How to read it

- §2 is the thesis in three cases: exit codes do not tell you whether the work got done — and
  neither does the gate, unless you decide which of the two is allowed to be wrong.
- §3 groups the nineteen items: gate semantics · assertion self-deception · time and content
  identity · detectors and orchestration · attestation, isolation and observation.
- §4–§6 are the cross-machine traps, the layered design, and the migration checklist.
- Appendices: all nineteen items in one table, plus a vocabulary note.

## What this is — and what it is not

- **Is:** one team's notes from months of running this pipeline in production for real work. Every
  failure described was found the expensive way — by shipping, then discovering that a green gate
  meant nothing.
- **Is not:** a framework, a library or a benchmark announcement. There is no code in this
  repository. The scripts referenced in the migration checklist are described, not shipped.
- **Evidence:** the numbers and log excerpts come from that pipeline's own round records,
  gate output and attestations. Those records carry internal topology and are not published; this
  text is redacted — no node addresses, hostnames, credentials, or internal system names.

Corrections and counter-examples are welcome via issues.

## License

Text: **CC BY 4.0** — see [LICENSE](LICENSE). Attribution: `pzmmy`.
