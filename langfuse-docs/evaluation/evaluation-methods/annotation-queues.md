---
title: Annotation Queues
description: Manage your annotation tasks with ease using our new workflow tooling. Create queues, add traces to them, and get a simple UI to review and label LLM application traces in Langfuse.
---

# Annotation Queues [#annotation-queues]

Annotation Queues are a manual [evaluation method](/docs/evaluation/core-concepts#evaluation-methods) which is build for domain experts to add [scores](/docs/evaluation/evaluation-methods/data-model) and comments to traces, observations or sessions.

## Why use Annotation Queues?

- Manually explore application results and add scores and comments to them
- Allow domain experts to add scores and comments to a subset of traces
- Add [corrected outputs](/docs/observability/features/corrections) to capture what the model should have generated
- Align your LLM-as-a-Judge evaluation with human annotation

## Set up step-by-step


### Create a new Annotation Queue

- Click on `New Queue` to create a new queue.
- Select the [`Score Configs`](/docs/evaluation/experiments/data-model#score-config) you want to use for this queue.
- Set the `Queue name` and `Description` (optional).
- Assign users to the queue (optional).


> ℹ️ **Note:** An Annotation Queue requires a score config that defines the scoring dimensions for the annotation tasks. See [how to create and manage Score Configs](/faq/all/manage-score-configs#create-a-score-config) for details.


### Add Traces, Observations or Sessions to the Queue

Once you have created annotation queues, you can assign traces, observations or sessions to them.

To add multiple traces, sessions or observations to a queue:

1. Select Traces, Observations or Sessions via the checkboxes. 
2. Click on the "Actions" dropdown menu
3. Click on `Add to queue` to add the selected traces, sessions or observations to the queue.
4. Select the queue you want to add the traces, sessions or observations to.


![Annotate](/images/docs/add_multiple_items_to_queue.png)

To add single traces, sessions or observations:

1. Click on the `Annotate` dropdown
2. Select the queue you want to add the trace, session or observation to


![Annotate](/images/docs/add_to_queue.png)
### Process Annotation Queue

You will see an annotation task for each item in the queue.

1. On the `Annotate` Card add scores on the defined dimensions
2. Click on `Complete + next` to move to the next annotation task or finish the queue


## Manage Annotation Queues via API

You can manage annotation queues via the [API](https://api.reference.langfuse.com/#tag/annotationqueues/GET/api/public/annotation-queues). This allows for scaling and automating your annotation workflows or using Langfuse as the backbone for a [custom vibe coded annotation tool](/blog/2025-11-25-vibe-coding-custom-annotation-ui).