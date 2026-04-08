---
title: Roadmap
description: Sneak peak into upcoming new features and changes in Langfuse. This page is updated regularly.
---

# Langfuse Roadmap

Langfuse is [open source](/open-source) and we want to be fully transparent what we're working on and what's next. This roadmap is a living document and we'll update it as we make progress.


> ℹ️ **Note:** **Your feedback is highly appreciated**. Feel like something is missing? Add
>   new [ideas on GitHub](/ideas) or vote on existing ones. Both are a great way
>   to contribute to Langfuse and help us understand what is important to you.


## 🚀 Released

export const ChangelogList = () => (
  <ul >
    {changelogSource
      .getPages()
      .sort(
        (a, b) =>
          new Date(String(b.data.date || 0)).getTime() -
          new Date(String(a.data.date || 0)).getTime()
      )
      .slice(0, 10)
      .map((page) => (
        <li
          id={page.url.replace("/changelog/", "")}
          key={page.url.replace("/changelog/", "")}
        >
          
            <span>{page.data.title}{page.data.date
              ? `(${new Date(String(page.data.date)).toLocaleDateString(
                  "en-US",
                  {
                    year: "numeric",
                    month: "short",
                    day: "numeric",
                    timeZone: "UTC",
                  }
                )})`
              : null}
          </li>
      ))}
  </ul>
);

10 most recent [changelog](/changelog) items:

> ℹ️ **Note:** Subscribe to our mailing list to get occasional email updates about new features.
> 
> ## Active Development

### Agent Observability

- Improve Langfuse to dig into complex, long running agents more intuitively

### Evals

- Introduce experiments as a first class citizen, remove the dependency on datasets to allow for more bespoke unit-tests
- Overhaul experiment (dataset run) comparison views to make it easier to work with experiment results
- Dataset management: bulk add traces to datasets
- Improve comments across the product to allow for more qualitative evaluation workflows and collaboration

### Playground

- Experiment with prompts/models in playground based on logged traces and datasets with reference inputs
- Langfuse model input/output data schema to increase model interoperability for structured outputs and tool calls
- Make Playground stateful and collaborative

### UI/UX

- Improve onboarding experience
- Improve core screens, especially for new and non-technical users
- Increase UI performance for extremely large traces and datasets

### Infrastructure / Data Platform

- We strongly increase ingestion throughput, response times, and error rates across APIs by simplifying the core data model.
- Move to an observation-only and immutable data model as it better aligns with complex agents and allows us to scale our platform. Thereby, we remove traces as a first class citizen.
- Improvements across our tracing UI to make it easier to find relevant spans for complex agents.
- Webhooks for observability and evaluation events, useful for routing and alerting

## 🙏 Feature requests and bug reports

The best way to support Langfuse is to share your feedback, report bugs, and upvote on ideas suggested by others.

### Feature requests

### Bug reports

} />

