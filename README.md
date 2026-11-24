# Multilingual Repair-Report Intake Assistant

Classroom prototype for AI for Social Good (SDG 11: Sustainable Cities and Communities). Team 18: Afshawn, Emran.

We used the Gemini API in one Colab notebook (prototype.ipynb). All reports are synthetic. No real residents, addresses or city systems are involved, and no API keys are in this repository.

## Problem

Persona (fictional): Mai, 68, lives in a San Jose apartment complex. She writes mostly in Vietnamese and reports a pile of dumped furniture by text message. English-only web forms with dropdowns for category and department are hard for her. When she writes freely, intake staff must read each message, work out the language, find the location, pick a category and reply. Reports with no location or an unclear category can sit unanswered, and a resident like Mai gets no response.

Simpler alternative we considered: a bilingual form with required location and category dropdowns. It is cheaper and more predictable. We chose free-text intake because the people hardest to serve (older residents, non-English speakers, people with low digital access) tend to write free text by SMS or email, and a form puts the burden on them. The tradeoff is that free text needs validation, described under Oversight and Tradeoff.

## AI Capability

Lab 2 (structured extraction) turns a free-text report into a fixed set of fields: language, location_text, issue_type, urgency, safety_hazard and confidence. It fits because a work queue needs the same fields every time.

Lab 1 (text generation) drafts a short acknowledgement, or a request for the address, in the resident's own language. It fits because extraction already tells us the language and what is missing.

Language, location and issue type are extracted from the text. Urgency, hazard and confidence are inferred by the model and are never trusted alone. The department is not chosen by AI; it comes from a local lookup table.

## Workflow

Synthetic report, then API call 1 (extraction) returns a JSON record. Local validation checks the format and that the location appears in the original text. Local routing then sends the record to the department queue (department_queue.csv) or the human review queue (human_review_queue.csv). API call 2 drafts a reply in the resident's language, except for safety hazards and unsupported languages. Intake staff review every draft before anything is sent.

Stays in Colab: the schema, validation, routing rules, department table and CSV queues. Sent through the API: the report text (call 1), and the report text plus the extracted issue and location (call 2). Returned: JSON matching the schema, and a draft message. Next actor: intake staff. Every draft is labeled DRAFT - staff must review.

Screenshots:

![Queues](screenshots/queues.png)

![Retest table](screenshots/retest_table.png)

## Edge Case Test

These results come from our own Colab runs (model gemini-3.5-flash-lite, 8 synthetic reports plus 3 additional tests). The numbers and outputs in the notebook are from our final run. See sections 5, 7c and 8 of the notebook.

Elicitation: we asked Gemini for six inputs likely to break the system. We treated its suggestions as candidate tests only, and wrote each expected route ourselves before running.

Failure chosen: T5, a gas smell. The report was "Strong gas smell outside the apartments at 100 Elm Court, I feel dizzy." The expected route was human review.

What v1 did: the model returned issue_type water_leak, urgency high, safety_hazard true and confidence 0.5. The model did flag the hazard, but the v1 routing ignores that field and only rejects confidence below 0.5, so the report was auto-queued to Water Utilities. The schema has no gas or hazard category, so the model had to choose the nearest wrong label. If wrong, this harms a resident in possible immediate danger whose report waits in a routine repair queue.

Change: v2 adds a local routing rule that sends any report with safety_hazard true to human review with no AI-drafted reply, and it defines safety_hazard in the prompt. v2 also requires a location that appears verbatim in the report, a clear category, a supported language and confidence of at least 0.7.

Retest: v1 passed 4 of 8 and v2 passed 8 of 8. v1 auto-queued T4 (no location), T5, T6 (vague report) and T7 (Amharic). v2 sent all four to human review, and the ordinary reports T1 to T3 and the injection test T8 were not over-flagged. In the run shown in our screenshots, the checks that caught them were missing_location (T4), safety_hazard (T5), unclear_category (T6) and unsupported_language (T7).

Remaining risks: the model's output for T6 changed between our two runs. In an earlier run it chose illegal_dumping with confidence 0.5, and only the confidence threshold sent it to review (low_confidence). In the final run it chose other_unclear and the category rule caught it. The route was the same both times, but the reason depended on run-to-run model variation, even at temperature 0. In T7 the model reported confidence 0.95 and labeled the Amharic report a pothole, so its confidence was misleading. The review queue still shows a department (Water Utilities) for T5.

Additional tests: we ran three more suggestions from the elicitation (T9 to T11). The elicitation prompt gives different suggestions each time, and these three came from an earlier run of it. All three went to human review, but all three were routed by the safety_hazard rule, which runs first. T9 and T10 would also have been caught by the location and category checks. T11 (an all-caps panic report) returned a pothole with confidence 0.95 and a valid location, so only the model's own hazard flag sent it to a person. The panic wording may have triggered that flag, and three tests cannot tell us whether the hazard was real.

Limit: eight synthetic reports in one run do not establish reliability. The retest only shows that these cases now behave as we expected.

## Oversight and Tradeoff

Human review happens in two places: for every record routed to human review (missing or ungrounded location, unclear category, safety hazard, unsupported language, low confidence), and before any drafted message is sent. Intake staff run the system. The AI supports a human decision and does not replace it.

Main ethical risk: a safety hazard, such as a gas smell, treated as a routine repair request, or an incomplete report silently queued so the resident is never helped.

Tradeoff: the added rules send more items to staff, which costs time, and some correct reports will be flagged. Half of our test set went to the review queue. Self-reported model confidence is a weak signal, which is why we use local checks. The location check can also flag correct reports if the model translates a place name instead of copying it.

## How to Rerun

1. Open prototype.ipynb in Google Colab. 2. Add GEMINI_API_KEY in Colab Secrets and enable notebook access. 3. Choose Runtime, then Run all (about 5 minutes because of free-tier pacing). 4. Check the outputs for keys or personal details before committing.

## AI Assistance

The notebook was drafted with Claude and then run, checked and edited by the team in Colab. We changed the model list after Google retired the original models, ran all tests ourselves, and wrote the findings from our own output.
