---
id: PFEC-001
title: AI Calling Solution — Technical Deep-Dive for Client Presentation
client: PFEC Global
status: in-progress
created: 2026-07-24
target: Client-facing meeting, week of 2026-07-27
---

# PFEC-001 — Technical Deep-Dive Expansion

## Background

Initial vendor-agnostic proposal (`deliverables/PFEC_AI_Calling_Proposal.docx` /
`.pptx`) is complete and covers architecture, feasibility, roadmap, and cost.
Client meeting next week needs a deeper technical round to build confidence
in the recommended approach.

## Scope

1. **End-to-end step detail** — expand each step of the call flow (lead
   eligible → Zoho webhook → middleware → voice platform → webhook
   write-back → downstream trigger) from summary bullets into full technical
   detail: request/response shapes, what Zoho config is needed, what the
   middleware does at each hop.
2. **Simple demo website** — a visual, non-production demo to make the
   concept tangible in the meeting (see open question below on exact form).
3. **Deeper framework/vendor comparison** — expand the 6-platform feasibility
   matrix (Vapi, Retell AI, Bland.ai, Bolna AI, Gnani.ai, full custom build)
   with more technical detail per candidate: integration depth, docs
   quality, concrete pros/cons, not just the current checkmark table.
4. **New slide(s): manual implementation walkthrough** — a build-runbook
   style slide (or slides) showing concretely how the solution gets built,
   step by step.

## Out of scope

- Cost estimation — already covered in the existing proposal; not part of
  this round.

## Acceptance criteria

- [ ] Call-flow steps have full technical detail, not just summary bullets
- [ ] Demo website exists and demonstrates the concept
- [ ] Framework comparison has expanded technical depth per candidate
- [ ] New implementation-walkthrough slide(s) added to the deck
- [ ] No cost content added in this round
