# AI Email Router with Amazon Bedrock Flows

**By Biswajit Boro** · July 2026

An email automation project built on Amazon Bedrock Flows. The flow reads an
incoming customer email, classifies its intent using a Bedrock prompt, and
routes it to one of two response-generation prompts depending on that
classification — producing a ready-to-send reply.

## What it does

1. An email comes into the flow as plain text.
2. An **EmailClassifier** prompt (running on Amazon Nova 2 Lite) reads the
   email and outputs exactly one label: `complaint`, `question`, or `refund`.
3. A **Condition** node (`ComplaintChecker`) checks that label. If it's
   `complaint`, the flow routes to a dedicated complaint-response prompt. Any
   other classification falls through to a general response prompt.
4. The matched response prompt generates a short, professional reply, which
   the flow returns as output.
5. A **Guardrail** is attached to the classifier node to filter harmful
   content (sexual content, violence, hate speech, and similar categories)
   before it reaches the model.

## Architecture

```
Email input
    │
    ▼
EmailClassifier (Prompt) ──► ComplaintChecker (Condition)
                                   │
                     ┌─────────────┴─────────────┐
                is_complaint                  default
                     │                             │
                     ▼                             ▼
      ComplaintResponsePrompt          GeneralResponsePrompt
                     │                             │
                     ▼                             ▼
              Flow output                   Flow output
```

## Building it

**1. Setting up Amazon Bedrock**
Logged into the AWS console, set the region to `us-east-1`, and confirmed
Amazon Nova 2 Lite was available as an inference profile.

**2. Building the classifier prompt**
Created a prompt in Prompt Management that reads a customer email and returns
exactly one word: `complaint`, `question`, or `refund`. Getting the model to
return *only* the label — with no explanation or markdown — turned out to be
harder than expected (see Debugging below).

**3. Building the response prompt library**
Created two separate response-generation prompts: one for complaints
(empathetic tone, acknowledges the issue, offers a resolution), and one for
general emails (brief, professional, neutral tone). Different email types
need differently shaped replies, so keeping them as separate prompts made
routing straightforward.

**4. Building the flow**
Wired everything together in Bedrock Flow builder: input node → classifier
prompt → condition node → two response prompts → two flow outputs. The
condition node reads the classifier's `modelCompletion` output and checks it
against `"complaint"`; anything that doesn't match falls to the default
(general) branch.

**5. Testing and debugging**
Tested the flow with several email types — complaints, questions, refund
requests, and a deliberately ambiguous email — using Bedrock's built-in trace
viewer to inspect exactly what each node received and produced.

**6. Adding guardrails**
Attached a Bedrock Guardrail to the classifier node with content filters
enabled for sexual content, violence, hate speech, and related categories.
Verified it by sending both a normal email (got a proper reply) and a
harmful-content email (the guardrail blocked it before either response
prompt could run).

## What actually went wrong (and how it got fixed)

The first version of this flow *looked* like it was routing correctly in
testing, but it wasn't. Two separate bugs were stacked on top of each other:

- **The condition expression was effectively always true.** It read
  `(conditionInput == "complaint") or (conditionInput != "")` — meaning any
  non-empty classifier output satisfied the "is complaint" branch, so every
  email routed to the complaint response regardless of its actual
  classification. Fixed by simplifying the condition to just
  `conditionInput == "complaint"`.
- **The classifier wasn't actually returning a clean label.** Even with
  explicit instructions to respond with "only the classification label,
  nothing else," the model was adding markdown formatting and a full
  explanation alongside the label (e.g. `**complaint**\n\nThe email
  expresses...`). Because the condition was doing an exact string match, this
  extra text meant it never matched `"complaint"` cleanly — a second,
  independent reason the routing was broken underneath the first bug.

Both bugs were only visible by pulling each node's individual input/output
trace during a test run, rather than just looking at the final reply — the
final output could look plausible while the actual routing logic underneath
it was broken. I used Claude to help interpret the trace output and pinpoint
exactly where the string match was failing, then rewrote the classifier
prompt to be far more explicit about output format (no markdown, no
punctuation, no explanation) as a fix, and tightened the condition to protect
against similar formatting drift.

## Files in this repo

- `flow-definition.json` — exported flow definition (nodes, connections,
  guardrail reference) via AWS CLI.
- `prompts/email-classifier.json` — the classifier prompt.
- `prompts/complaint-response.json` — complaint response generator.
- `prompts/general-response.json` — general/default response generator.
- `screenshots/` — flow diagram and test trace screenshots from the Bedrock
  console.

## Tools and concepts

**Tools used:** Amazon Bedrock Flow builder, Bedrock Prompt Management,
Bedrock Guardrails, AWS CLI, Amazon Nova 2 Lite.

**Concepts learned:** conditional routing logic, prompt output formatting and
why models don't always follow "respond with only X" instructions as
strictly as expected, reading Bedrock's node-level execution traces to debug
a multi-step flow, and configuring content-safety guardrails on a model node.

## Time and effort

This took about 4 hours in total. The classifier was the most time-consuming
part — the initial prompt looked correct and matched the instructions I'd
been given, but the routing still didn't behave as expected. Pulling the
trace output for each node and working through it (with some help from
Claude to interpret what the trace was actually showing) revealed that the
model's output format, not just the condition logic, was the real problem.
Once the prompt was rewritten to constrain the output strictly to a single
word, the flow worked correctly across all tested email types.
