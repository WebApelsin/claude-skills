---
name: grilling
description: Interview the user relentlessly about a plan or design. Use when the user wants to stress-test a plan before building, or uses any 'grill' trigger phrases.
---

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one.

Ask one question at a time and wait for my answer before continuing — asking several at once is bewildering. Drive each question through the AskUserQuestion tool (one question per call) so I can pick an answer or write my own.

For every question, propose 2–4 concrete candidate answers as the options. Ground each one in something real and say briefly why it's a contender and its tradeoff:
- the existing codebase — cite the file, pattern, or convention that justifies it;
- an established best practice;
- or plain common sense when neither applies.

Put your recommended option first and append "(Recommended)" to its label. The tool always adds an "Other" choice, so I can hand-craft my own answer to any question — don't restate that in the options.

If a question can be answered by exploring the codebase, explore it instead of asking me.
