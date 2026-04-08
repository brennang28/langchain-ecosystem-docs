---
title: Scores via UI
description: Annotate traces and observations with scores in the Langfuse UI to record human-in-the-loop evaluations.
sidebarTitle: Scores via UI
---

# Manual Scores via UI

Adding scores via the UI is a manual [evaluation method](/docs/evaluation/core-concepts#evaluation-methods). It is used to collaboratively annotate traces, sessions and observations with evaluation scores.

> ℹ️ **Note:** You can also use [Annotation Queues](docs/evaluation/evaluation-methods/annotation-queues) to streamline working through reviewing larger batches of of traces, sessions and observations.


## Why manually adding scores via UI?

- Allow multiple team members to manually review data and improve accuracy through diverse expertise.
- Standardized score configurations and criteria ensure consistent data labeling across different workflows and scoring types.
- Human baselines provide a reference point for benchmarking other scores and curating high-quality datasets from production logs.

## Set up step-by-step


### Create a Score Config

To add scores in the UI, you need to have at least one Score Config set up. See [how to create and manage Score Configs](/faq/all/manage-score-configs) for details.

### Add Scores

On a Trace, Session or Observation detail view click on `Annotate` to open the annotation form.


![Annotate](/images/docs/trigger_annotation.png)
### Select Score Configs to use


![Annotate](/images/docs/select_score_configs.png)
### Set Score values


![Annotate](/images/docs/set_score_values.png)
### See the Scores
To see your newly added scores on traces or observations, **click on** the `Scores` tab on the trace or observation detail view.


![Detail scores table](/images/docs/see_created_scores.png)

## Add scores to experiments

When running [experiments via UI](/docs/evaluation/experiments/experiments-via-ui) or via [SDK](/docs/evaluation/experiments/experiments-via-sdk), you can annotate results directly from the experiment compare view. 


> ℹ️ **Note:** **Prerequisites:**
> - Set up [score configurations](/faq/all/manage-score-configs) for the dimensions you want to evaluate
> - Execute an [experiment via UI](/docs/evaluation/experiments/experiments-via-ui) or [SDK](/docs/evaluation/experiments/experiments-via-sdk) to generate results to review


![Annotate from compare view](/images/changelog/2025-10-23-annotate-compare-view-overview.png)

The compare view maintains full experiment context: Inputs, outputs, and automated scores, while you review each item. Summary metrics update as you add annotation scores, allowing you to track progress across the experiment.

## GitHub Discussions

