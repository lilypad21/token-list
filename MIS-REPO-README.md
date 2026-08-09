# Mueller Investigative Services — Contracts

Working repository for engagement agreement drafting. Holds the MIS agreement
template, per-matter intake material, and the drafting agent.

## Layout

```
.claude/agents/
  mis-contract-drafter.md      the drafting agent

templates/
  mis-agreement-template.md    the MIS engagement agreement template

matters/
  <matter-slug>/
    notes/                     intake notes, call notes, consultation records
    emails/                    scope-of-services correspondence
    background/               client org, underlying dispute, prior engagements
    agreement-DRAFT-<date>.md  agent output
```

One folder per matter. Slug it something short and non-identifying —
`aim-2025-06` rather than a client name plus the nature of the allegation.
Folder names show up in commit messages, terminal output, and anywhere the
repo is browsed.

## Running the agent

Drop the matter's material into `matters/<matter-slug>/`, then ask for a draft
against that folder. The agent will:

1. Read the template first and inventory every field it must fill
2. Read the notes, emails, and background
3. Reconcile conflicts between sources
4. Fill the template — preserving its legal language verbatim
5. Return the draft plus a source map, open questions, and risk flags

It opens with a count: **"N fields need input, M conflicts found, K risk
flags."** That is the triage line — it tells you how much work the draft still
needs before it goes anywhere.

## What the agent will not do

- Invent a term. Unsourced fields come back as `[[NEEDS INPUT: field]]`, never
  as a plausible guess.
- Rewrite the template's legal language. It fills variables; it flags clauses
  it thinks are wrong rather than editing them.
- Produce anything final. Every output is marked **DRAFT — FOR INTERNAL
  REVIEW** and assumes a human reads it before a client does.
- Send anything. It writes a file and hands it back.

## Before the first run

The template needs to be in `templates/`. The agent stops without it by
design — it will not fall back to a generic contract form, because a
generic-form agreement carrying the MIS name is worse than no draft at all.

## A note on what lives here

This repository accumulates client intake material: who is under
investigation, what they are alleged to have done, and what the client is
willing to pay to find out. Some of it may be attorney-directed and
privileged.

Keep the repository **private**. Keep collaborator access to people who would
already be inside the matter. If work is attorney-directed, confirm with
counsel whether the material should live in a shared repository at all, or
stay within their document system — putting privileged work product in a
third-party repo is a question worth asking once, deliberately, rather than
discovering later.
