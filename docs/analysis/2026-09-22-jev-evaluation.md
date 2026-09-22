# Jev evaluation — 22 September 2026

Jev looks worth trying for executor-question routing. It matched our labels on all 12 routing examples, but this was a small synthetic test. We have not measured whether it saves tokens, time, or money compared with the director. This note records the experiment; it does not add an integration.

## What we tested

We called `typesafe-ai/jev` through Vercel AI Gateway's evaluation endpoint for two tasks: tagging engineering feedback and deciding where an executor question belongs. The [fixtures and rubrics](2026-09-22-jev-fixtures.json) contain 12 examples per task. Expected labels and rubrics were fixed before the run. Each case was submitted twice, sequentially, for 48 successful calls.

The labels are the author's judgments. They were not independently reviewed, and some feedback categories overlap. These are synthetic examples written for this experiment, not a held-out sample of real director work.

Routing covered technical facts, routine engineering choices, business decisions, merge approval, mixed questions, and missing context. Two cases across the dataset included instructions to override the classification policy. Both matched their expected labels; that is too little evidence to claim general resistance to prompt injection.

## Results

| Measure | Observed result |
| --- | --- |
| Question routing | 12/12 unique cases; 24/24 calls matched expected labels |
| Feedback tagging | 11/12 unique cases; 22/24 calls matched expected labels |
| Same choice on both repetitions | 24/24 cases |
| Successful-attempt latency, median / p95 | 392 / 615 ms |
| Input / output tokens | 25,856 / 3,596 |
| Gateway market cost | $0.001086 for 48 successful calls |
| Gateway reported cost | $0 for those calls |
| Accepted at chosen-option probability ≥ 0.99 | 28/48 calls (58%), 26 correct; both mistakes still accepted |

Latency includes network time but excludes pacing and rate-limit backoff. The gateway's own provider timing puts the median at 236 ms; the rest is network and gateway overhead. Three HTTP 429 responses were observed during the run. The stored results contain one of them; the other two were retried inside the runner and appear only in console output that was not kept, so that count cannot be checked from the artifacts. Requests resumed with pacing and backoff, so the latency figures do not describe total elapsed time or sustained throughput. Cost and token totals exclude two separate connectivity probes. Gateway cost metadata is not a billing statement, and the reported $0 is not a promise of free usage. The market cost equals the listed price: $0.042 per million input tokens, with output not charged.

The one mismatch happened twice:

> The runtime parser belongs at the incoming request, not in each internal helper.

We expected `types_and_schemas`; Jev chose `organisation`, with a choice probability of 0.99 and provider confidence of 0.98. The sentence concerns both placement and boundary validation, which exposes overlap in the taxonomy. Under our frozen labels it is a mistake, and a high-confidence cutoff would have accepted it. Agreement across two repetitions does not establish accuracy or calibrated confidence.

## Repeating the experiment

Use the companion JSON without changing its labels or rubrics. For each of two passes, submit every case in file order. The request body is `{"state":{"text":case.text},"questions":{"decision":questions[case.task]}}`; compare `answers.decision.choice` with `case.expected`.

The run used `POST https://ai-gateway.vercel.sh/v4/ai/evaluation-model` with bearer authentication, JSON content type, and these headers:

```text
ai-gateway-protocol-version: 0.0.1
ai-evaluation-model-specification-version: 4
ai-model-id: typesafe-ai/jev
```

After throttling, requests used 3.2-second pacing and bounded retries following `Retry-After`. A repeat should record every failed attempt and total elapsed time as well as successful-attempt latency. The model alias was not pinned to an underlying version, so later runs may differ. The public fixtures allow a fresh comparison; aggregate results here are from the original run, whose raw gateway responses are not included.

The [Vercel Jev model page](https://vercel.com/ai-gateway/models/jev) and [gateway evaluation adapter](https://github.com/vercel/ai/blob/main/packages/gateway/src/gateway-evaluation-model.ts) are the vendor references for the model and request interface.

## What we should do next

Run Jev alongside the director on real executor questions, showing its routing suggestion without letting it dispatch work. Freeze the rubric, have the director label examples before seeing Jev's answer, and compare disagreements against the current workflow. Measure total latency including retries, token use, and cost for both paths. That comparison should decide whether an integration is worthwhile.

Keep approval, verification, and state-transition gates deterministic. A suggested route cannot authorize a merge or an irreversible action. On API errors or invalid answers, use the existing director workflow; leave mixed and unclear requests with the director. Keep feedback tags advisory until the taxonomy and its boundary cases have had a separate evaluation.

A future integration would need requirements and married tests through `/lazyspec`. This experiment changes no software behavior or requirement headings, and does not justify a global taste card.
