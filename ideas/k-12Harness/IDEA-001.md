---
id: IDEA-001
date: '2026-09-26'
by: shane
version: 0.5.0
commit: 50a2869a6caf
status: new
priority: null
reason: null
reviewed: null
evidence:
- work/20260908T113155
---

Isolated intent-stage grading (evals/live.py --stage intent) grades only question classification (grade_intent_question_match). StructuredIntent also carries excluded_topic, which Stage 1 alone decides for should_refuse cases. Consider a grader for excluded_topic on refusal cases, with a load-bearing mutation (ADR-013) and a golden-case field for the expected topic. Owner decision in WI 20260908T113155: question classification only for that WI; this is a separate, small WI.
