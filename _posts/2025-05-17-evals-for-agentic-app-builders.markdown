---
title: Creating Meaningful Evaluations for Agentic App Builders
layout: default
date: 2025-05-17
keywords: evals
published: true
mathjax: yes
---

For the last two months at [Create](https://www.create.xyz), my main focus has been building evaluations for our AI agent -- the thing that takes a user's natural language prompt and turns it into a working application. This is a hard problem, and the space is largely unresearched, so I want to share what we've learned so far.

# What are evaluations?

Evaluations are automated setups where we give the AI agent a set of tasks (an array of prompts that a user would have typed in manually), and then score how well the agent performs on those tasks. Scores range from 0% to 100%, and there can be multiple scores per run. The key property of a meaningful score is that it correlates with quality: a higher score than before implies improvement, a lower score implies regression.

In simpler terms, it is a way to give the AI agent grades on various tasks. If the grade changes day-over-day after our engineers ship code changes, it tells us whether the product experience got better or worse.

# Why this is hard

Traditional software tests assert exact outputs. If `add(2, 3)` returns `5`, it passes. Agentic systems don't work this way. The prompt "build a mentoring marketplace with Stripe payments" has no single correct answer, and the agent could produce many valid implementations that all look completely different.

We tried the common approaches, and each has limitations.

**LLM-as-a-judge**: pass the output of the agent to another LLM and ask it to score the result. The problem is that LLMs hallucinate success and miss functionality bugs when reading codebases. They have no way to actually _run_ the code or test the web app.

**Computer Use Agents (CUA) as judge**: a computer use agent like OpenAI's Operator uses a physical browser to interact with the generated website and judge it. This is better because it actually tests the app, but accuracy is still not where it needs to be. Anthropic's CUA performs at around 90% accuracy on our text-to-app evaluation set, while OpenAI's Operator is at around 70%. These error rates get embedded into the provided scores.

**String or DOM comparisons** break whenever the agent chooses a different but valid layout.

**Manual QA** doesn't scale.

# What we built

We built a daily evaluation framework that simulates user sessions, scores the results, and gives engineers a signal on whether the agent is getting better or worse. It has three main components.

**Prompt sets** are sets of prompts drawn from real user sessions, edge cases, and synthetic flows. We have two main categories:

- **Text-to-app one shot**: Given a single prompt, how well can the agent build the app in one go?
- **Long conversation**: Given a long array of prompts in series, how well can the agent implement all of them without introducing regressions?

Each set holds roughly 1 to 50 high quality examples.

**Scorers** calculate the scores. We use three types:

1. **LLM judges** for relevance and coherence.
2. **Computer use agents** that launch the generated app in a real browser, click through flows, and observe state.
3. **Humans-in-the-loop** for side-by-side comparisons on selected failures.

The human-in-the-loop component is important. Because no automated method is 100% accurate, gold standard evaluation sets must be feasible for human experts to evaluate. There must be a small enough number that humans can review day-over-day, and they need to provide meaningful signal on each run.

**Daily tracking**: Scores, logs, and qualitative notes go into [Braintrust](https://www.braintrust.dev/). Engineers can review regressions in minutes. If a change lands, we see the effect the same day.

# Scoring vs meaningfulness

Balancing accurate scoring with meaningfulness is difficult.

You can build automatable evaluations by forcing the agent to produce specific, testable outputs -- for example, requiring hardcoded `data-testid` attributes on page elements so you can write deterministic assertions. These are stable and can run without human review.

But such evaluations don't represent what a real human would type. No real user prompts "build me a landing page and make sure the hero section has `data-testid='hero-section'`." Evaluations like these don't tell you how the product experience is changing.

We went with evaluations that represent real human use, even though it means we need humans in the loop and can't fully automate scoring. Selecting the right set of evaluation examples requires a lot of experimentation.

# What we've learned

Evals have been useful for identifying issues with the agent. By their nature, they are a small set of high quality examples that provide a lot of information on each run. The small size allows for humans to build intuition around the behavior of the agent (which can be noisy due to the fundamental nature of LLMs), and the fact that the evals run repeatedly allows engineers to identify persistent issues as well as flaky ones. Some concrete results:

- We caught regressions before they hit users.
- We flagged prompt templates that degraded output quality.
- We stretched one long conversation evaluation from 8 to 165+ turns without errors, by iterating on the eval and using it to drive improvements.


# Where this fits in the bigger picture

Create's platform produces full applications for non-technical users from natural language. That includes auth, payments, databases, deployment, integrations, etc. The agent has to be reliable, and reliability requires metrics and feedback loops.

Evaluations provide information about the current state of the platform independent of production data. This allows us to see how configuration changes to things like models or tools will affect performance.

However, these evaluations are still limited in scope. The ultimate test is how the agent performs across real production sessions.

# Industry context

Academic benchmarks like SWE-Bench, WebArena, and AppBench focus on code reasoning or single-turn tasks. They don't cover the full workflow of generating a working app from a prompt, which is what we need to evaluate. As far as I can tell, there isn't much existing work on evaluations for text-to-app agents specifically.

# Conclusion

Evaluations are a constant work in progress. The question "how well does a generated website represent a prompt?" is still open-ended, and there is no state of the art way to answer it fully automatically. But even imperfect evaluations have already helped us catch real issues, and I expect them to keep being useful as the agent gets more capable.
