# Presenter Notes & Delivery Guide

Tips for the workshop facilitators on delivery, timing, and common pitfalls.

---

## General Tips

- **Pair-present:** One person drives slides/theory, the other live-codes demos.
  During labs, both circulate to help participants.
- **Live-code, don't just show slides:** For every module, have a working demo
  project ready. Code along with the audience for at least the first lab.
- **Keep energy up:** The morning is theory-heavy — use frequent demos, polls,
  and "try it yourself" moments to keep engagement high.
- **Timebox strictly:** Labs can easily run over. Set a visible timer and give
  5-min and 1-min warnings.

## Module-Specific Notes

### Welcome & Intro
- Start with a compelling demo: show a fully working agent doing something
  impressive (e.g., a multi-tool agent booking a meeting + looking up data).
  Then say "by the end of the morning, you'll know how to build this."
- Architecture diagram should be printed / kept visible throughout the day.

### Module 1 — Agents Fundamentals
- **Common issue:** Azure OpenAI credentials. Ensure everyone has run `az login`
  and has the correct endpoint/deployment before starting the lab.
- Have a fallback: if Azure OpenAI isn't available, show how to swap in Ollama
  for local inference.
- Demo both streaming and non-streaming — the difference is visceral.

### Module 2 — Tools
- **This is the "aha moment" module.** When participants see the LLM
  autonomously decide to call their function, it clicks.
- Emphasize: good tool descriptions = good tool use. Show a bad description
  and how the LLM fails to call the tool properly.
- Agent-as-a-Tool is a stretch goal — cover it in theory, let advanced
  participants try it in the lab.

### Module 3 — Conversations, Memory & Middleware
- Session management is critical for production apps. Stress the importance of
  session IDs and state management.
- Middleware is enterprise-grade plumbing — security teams love it. Frame it
  as "how you make agents production-ready."

### Module 4 — Workflows
- This module bridges theory and hackathon. Show how workflows let you
  orchestrate what would otherwise be spaghetti code.
- If time is tight, focus on theory + a single live demo. The lab can be
  shortened to "observe and modify" a pre-built workflow rather than
  building from scratch.

## Hackathon Notes

- **Team formation:** If people don't know each other, do a quick icebreaker
  or assign teams based on skill mix (ideally each team has someone who
  completed all the labs comfortably).
- **Starter code:** Prepare integration points in the existing app ahead of time:
  - A service interface where an agent can be plugged in
  - A chat endpoint/UI component ready to wire up
  - Sample data the agent tools can query
- **Mentors:** Assign specific mentors to specific teams. Have a "floating"
  mentor for cross-team questions.
- **Mid-hack check-in (15:30):** Quick 2-min status per team prevents teams
  from going down rabbit holes. Redirect if needed.
- **Demo prep (16:15):** Remind teams to stop building and start preparing
  their demo. A working demo of 1 feature beats a broken demo of 3.

## Timing Buffer

The schedule has built-in buffer in two places:
1. Lunch (45 min) — can extend to absorb morning overruns
2. Hackathon (2.5 hr) — the mid-break can be shortened if the briefing ran over

If the morning runs long, cut Lab 4 (Workflows) to a guided walkthrough
rather than hands-on, and redirect that time to the hackathon.

## Post-Workshop

- Send follow-up email within 24 hours with:
  - Slide deck
  - Lab solutions (GitHub repo)
  - Link to official MAF docs and community
  - Feedback survey
  - Photos from demo presentations (if taken)
