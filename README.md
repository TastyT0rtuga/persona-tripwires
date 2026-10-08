# Three cheap tripwires for a multi-persona assistant

How we detect indirect prompt injection in a home assistant without asking
another model for an opinion — and the two times our own measurement lied to
us.

> Hobby project; defensive only. This is a design write-up, not a library.
> No code is released, and nothing here identifies the household it runs in.

---

## The problem

A household assistant runs about twenty personas on shared models. Each
persona reads things it did not write: web pages, mail summaries, device
names, documents. Any of those can carry text written to steer the persona
(indirect prompt injection). The usual answer is to ask another model whether
the input looks hostile. That adds cost and delay, and it puts a second model
in the path that can be steered the same way.

We wanted detection that:

- costs nothing per turn,
- cannot be argued with, because no model is asked for an opinion,
- produces a finding a tired person can act on in one line.

The principle throughout: **rules decide, models explain.**

## Tripwire 1: a decoy tool

Every persona that has tools is shown one extra tool that no household
request ever needs. Its description says so plainly. A persona that calls it
did so because something it read told it to.

- The call does nothing and returns a bland error.
- The persona loses its action tools for the rest of that turn.
- One finding is recorded: which persona, when, and what it tried to pass.

**Why it works:** an honest request never produces the call, so the
false-positive rate is close to zero by construction.
**Why it's limited:** an attacker who knows the decoy's name simply avoids
it. Pick your own name and description, and change them if they leak.

## Tripwire 2: a marker in each persona's private instructions

Each persona's instructions carry a short random-looking reference, different
per persona, derived from a key the personas cannot read. If that string
appears in a reply, or in anything a persona sends to a tool, the
instructions are being copied out. The string is withheld and the event
recorded. Because the marker differs per persona, a finding also says *whose*
instructions leaked.

**Limit:** it catches a verbatim copy. A paraphrase passes.

## Tripwire 3: plain-statistics fingerprints

A count of which tools each persona uses, and how often per hour. Two rules:

- a tool this persona has never used before,
- a burst several times above the persona's own busiest hour, over a floor.

Nothing is reported until a week of history exists.

**What went wrong first:** the first run treated every tool as "never used
before" and sent a false alert naming three personas. The fix was to make the
first pass silent. We kept the false rows, moved aside and labelled, rather
than deleting them.

## Keeping drills out of the baseline

We test our own personas with planted instructions every week. Drill traffic
is marked at the source and excluded from the fingerprints and from the real
findings file. Without that, the tests would teach the baseline that attacks
are normal, and every drill would page someone.

## What we measured

- **Weekly integrity sweep:** 75 planted tests across the personas. 70 clean;
  2 where the persona never reached the bait; 3 not run, because that
  persona's user had not opted in to testing. Cost about $1.22 and 4.5
  minutes per sweep. An all-clean result says more about the tests than the
  personas: these are the obvious versions of each attack, and current models
  resist obvious planted notes. Harder variants are the real work.
- **Our first measurement lie:** an earlier version of the sweep counted 26
  refusals by a permission gate as "clean". Those personas had never been
  tested at all. We added a "not run" outcome and re-ran. Reporting an
  untested thing as passed is exactly the failure this design is meant to
  prevent, and we did it.
- **Our second:** the fingerprint false alarm above.
- **Door tests:** eight separate attacks against the wider system. Two did not
  work, one was partly noticed, five went unnoticed. The three tripwires were
  built in response to those five.

## What this does not do

- It does not stop an attack that uses only the tools a persona normally
  uses, at a normal rate, without copying instructions.
- It does not judge content. A persuasive lie that leads to a normal-looking
  action passes all three.
- It is detection. Containment — what happens after a finding — is a separate
  layer.

## Prior work

None of the three ideas is new on its own; the point of this write-up is the
combination, the cost, and the honest numbers.

- **Decoy tools.** Honeypot tools exposed to agents over the Model Context
  Protocol do the same job as Tripwire 1: [HoneyMCP](https://github.com/barvhaim/HoneyMCP)
  adds "ghost tools" that no legitimate workflow should call, and
  [Beelzebub](https://github.com/mariocandela/beelzebub) offers MCP bait
  tools that make suspicious invocations observable.
- **Canary strings.** [Rebuff](https://github.com/protectai/rebuff) adds a
  canary word to prompts that should never appear in the output, to detect
  prompt leakage — the same mechanism as Tripwire 2. Ours differs only in
  being per persona and derived from a key the personas can't read.
- **Known-answer detection.** Liu et al., *Formalizing and Benchmarking
  Prompt Injection Attacks and Defenses* (USENIX Security 2024,
  [arXiv:2310.12815](https://arxiv.org/abs/2310.12815)), benchmark a related
  idea: append an instruction with a known answer and flag data that makes
  the model miss it.

## Open questions

- Does a per-persona marker placed in *bait*, rather than in instructions,
  tell you which persona passed tainted text along?
- Do personas treat machine-written bait differently from bait a person
  wrote?
- Does grading the source of everything a persona read (a reliability letter
  and a credibility number, as analysts do) predict which turns go wrong?
  This is running in shadow mode; no data worth reporting yet.

## License

CC-BY-4.0. Personal hobby project, not affiliated with or endorsed by any
employer.
