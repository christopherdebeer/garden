---
title: Contextual is a tool for making tools that do stuff
status: seedling
created: 2024-04-27
---

[[project]]

A tool for making tools [that do (stuff)]

https://www.tldraw.com/r/IP6jxGHDdiQPDz4zZLy7R?v=1489,-222,854,1745&p=ICvY4f1sAuDI5WAhacKn6

(As a software engineer I make tools that do stuff all the time, tools are meant [intended?] to enable people to do stuff that they might not otherwise have been able to do, or might which would have been a pain to do. Programming is a tool too, but in the same way that a hammer doesn't make everyone a master carpenter, programming allows people like myself to make tools for others, but has a recursive dependence on learning programming. My goal is to make a tool for making tools that doesn't require programming (in the traditional sense) ie without requiring the user to "learn to program".)

Frequently referred to as ctxl, and formerly known as prompt studio, which started off as an LLM chat chain interface intended to be repeatable and initially allowing for specific data elements to be used in the "prompts" cf. prompt templates.

[brief pause] … as it stands there is no doubt that with the right prompts and data inserted into the context, this is repeatable and eminently useful. However, the challenge is getting to the right configuration.

[[Contextual]] instead, starting from the practice adjacent to unit testing in software engineering of [[evaluations]] to quantify a score for a given task. Contextual (dog foods) embraces the generative capabilities of (at least at present) transformer based language models to adaptively guide the user from an initial "objective" to craft "measures" and iteratively refine tasks to produce data, outputs.

[Aside]: Evals ([[evaluations|Evaluations]]) of LLM prompted responses, which in an ideal case, allow for exacting scoring (empirically best when the measure is based on classification) when run against a usefully large dataset. And at least so far as my experience goes, very hard to create for non-trivial tasks.

[cont] We seem to be missing a trick, whereby if we grouped or otherwise labeled our measures, we would at face value be able to, at a more granular level, quantify the differences and in a sense dynamically weight influence on the differentiation of candidate prompts.

[/aside]

Affordances.

There are two categories of affordance first, the current [read gen ai boom] practice of [[tool-use]] or function calling for large language models, and the OG affordance given to the user often in the form of an interface or user interface (UI).

[appendix?] Potential flows

1. Starting with objective as the only exposed affordance (I know I know not chat first again) but bear with me.
2. Upon receiving this first interaction nominally an objective the system attempts to build up a "context"
3. Each new or edited "objective" resulting in the proposal and either persistence (not dismissed by the user) or iterative refining into a set of "tasks" taking input data (derived or user supplied) to generate output data.
4. This data as it pertains to a given task can have measures associated with it which form the basis of evaluating the current configuration in a quantitative manner.

https://overcast.fm/+NPG-Q1GJc
Elephant in the Room - Future of Coding

It surprises me that at this point I have not already gone on at length about [[malleability]]. cf. [[moldable-development]] (@feenk; Girba et als gtoolkit) and my presumption that malleability or adaptive and responsive interfaces are more than just needed for "programming" and perhaps a stretch but, if you have the malleability, the [[direct-manipulation]] (cf. [[brett-victor|Brett Victor]]) then in fact it is programming that has been enabled.

Prompts (not that kind)

In the introduction I spoke about the project that Contextual evolved from which has a very specific meaning for Prompt, however in this instance I mean it in the writing or creative sense, an affordance given, to the user, to help or guide them.

Intro prompt/affordance

[let's (
  Make a tool,
  Get something done,
  Have some fun,
  Explore,
  Experiment,
  Learn something,
)]

Generative prompts produced by the system during the course of iterating or interacting with it, are fed back to the user as various visual affordances. Suggesting a new data element or potential task, or improvements to those already existing.
