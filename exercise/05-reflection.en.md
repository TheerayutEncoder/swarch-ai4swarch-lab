**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:** Theerayut Attajak / 6970142021
**My role on the review board:** Architect

After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

After Parts A and B, I would trust AI with the groundwork of architecture: recovering structure from protos and manifests, drafting a complete first ADR, checking facts and logic, and generating ATAM-style findings. It did all of these quickly and usually cited its evidence. I would keep for humans the final decisions, the stakeholder knowledge nobody writes down, verifying every AI claim, and defending the design. My example is the double-points bug: the AI review noticed that a retried PlaceOrder gets a new order ID, so the de-duplication fails, which I had missed as the architect. But the review read only my ADR and the scenario, so it could not judge how much the bug mattered. The board decided it was a required change rather than a reason to reject, next to facts only stakeholders had, like the printed campaign. The AI found the bug; humans decided what it meant.
