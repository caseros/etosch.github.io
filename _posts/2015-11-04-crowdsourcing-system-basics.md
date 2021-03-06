---
layout: post
title: Crowdsourcing system basics
permalink: crowdsourcing_basics.html
date: 2015-11-04 11:39:59.000000000 -05:00
type: post
published: true
status: publish
categories:
- Research
tags: []
meta:
  _edit_last: '20775'
author:
  login: etosch
  email: etosch@cns.umass.edu
  display_name: Emma Tosch
  first_name: Emma
  last_name: Tosch
---
Some time ago I started writing up the requirements for a sound crowd sourcing system. Last year I wrote about Amazon's report on using verification tools at scale and it got me thinking about the kinds of abstractions one would need to formally verify a crowdsourcing system.

<!--summary-->

Before implementing any verification system, it's important to to know what the basic guarantees are -- what invariants we should define, so that we may define what it means for the system to be correct? Crowdsourcing is used in a variety of contexts, and different crowdsourcing systems serve different labor markets. In particular, I will focus on general human intelligence tasks and explicitly will not address specialized markets.</p>
\nIn particular, I would like to address three rough categories of tasks: point estimation tasks, distribution estimation tasks, and causal inference tasks.</p>


### Point estimation tasks
Point estimation tasks are those that have a human-verifiable single solution. They are the kinds of tasks that AutoMan is designed to handle. These tasks cover annotation tasks, data cleaning, and spammy tasks such as following a link to vote for something. The unifying feature of these tasks is that another human can evaluate them. These are the kinds of tasks that may eventually be performed by machines, but currently still require some curation by humans. Point estimation tasks use agreement to evaluate the quality of responses returned.

**Correctness**: If a task cannot be verified by a human curation (i.e., agreement cannot be reached), then the program is incorrect. The inherent truth of the underlying system allows us to use traditional statistical methods for convergence in the limit (e.g., hypothesis testing).

**Requirements**: The main requirements of a crowdsourcing system that handles these tasks are that there is no collusion between respondents and that each sample is unique. That is, each time the system requests a response, that response must come from a different person.

### Distribution estimation tasks
Distribution estimation tasks are those that may not converge to a point, but whose response distribution we expect to stabilize for a given point in time with sufficient data. In point estimation tasks, this would denote an error, which brings up an important distinction in how we model and reason about correctness for different crowdsourcing tasks: there is no single canonical criterion for correctness. How we classify tasks is critically important. It is possible that a researcher will consider a particular task to be a point estimation task, but in truth it is a distribution estimation task. Mis-classification is, of course, a bug, but it's a peculiar kind of bug -- one that can only be both caused and detected by a human. Distribution estimation tasks are the kinds of tasks that SurveyMan is designed to handle.

**Correctness**: This problem is a bit tricky -- in SurveyMan, we try to do as much offline analysis as possible, to bound our certainty of classification, given what we know about about the instrument used to collect data. In particular, when using surveys, questionnaires, or collections of polls to collect data, there may be subtle biases introduced via the framing of the problem and imprecision in natural language. The kinds of tasks completed and questions answered are understood to not have a "gold standard." These tasks are typically administered as collections of tasks or questions. The tasks or questions that comprise the single survey or questionnaire or otherwise complex task each have varying degrees of variability -- individuals may change their opinions over the course of a survey, they may change their country of residence between surveys, and they may change their gender over their lifetime. Each item we try to measure has some degree of impermanence and my current research attempts to use these distinctions to help with our analyses. In any case, the point of this section is that correctness for these tasks differs dramatically from correctness for point estimation tasks.

**Requirements**: We assume that these tasks have been designed for a general audience. The main requirements are that (1) respondents do not collude, (2) every qualified respondent is equally likely to have been exposed to the task (i.e., every qualified respondent has had the opportunity to accept or ignore the task), (3) the system permits an adaptive pricing strategy. Requirement (2) is a bit tricky for several reasons. Firstly, we need some way of gather demographic information. AMT has basic demographic information (most importantly for Amazon -- country of employment) and allows requesters to filter respondents via a qualifications mechanism. If the task requires qualifications that AMT cannot express (e.g., handedness), or if we are working on a system that does not have qualifications, the complex structure of the task allows requesters to write a "short circuit" into the task.

### Causal Inference Tasks
Causal inference tasks can be viewed as a combination of the previous two tasks. In some ways they are more easily described and tested than the others. However, they have other complications that will be instrumental in our requirements specification for crowdsourcing systems. Causal inference tasks are those that measure the effect of a particular treatment. The <a href="https://en.wikipedia.org/wiki/Stroop_effect">Stroop effect</a> is a classic observation from psychology that requires randomized exposure to two types of stimuli in within-subjects test. The infrastructure from AutoMan and SurveyMan can be used to express these tasks, with some additional logic provided by our in-progress project, ExperiMan. I have been meaning to look into how the <a href="http://www.cs.berkeley.edu/~kjamieson/resources/next.pdf">NEXT</a> <a href="https://github.com/nextml/NEXT">system</a> for active learning works because I think there might be some insights to be gained by this.

**Correctness**: The standard way to measure a causal effect is to run an experiment and to estimate the effect of counterfactual exposures. Estimation requires certain randomization prerequisites be met and that the quantity being estimated be congruent with the assignment procedure.

**Requirements**: The main requirement is that assignment to treatment and control is unbiased. The unbiased guarantee should be similar to the one for point estimation. We would also like to be able to do some more sophisticated assignment procedures that would be difficult to manage in standard crowdsourced experiments, such as conditional random assignment on the basis of demographic information learned over time. One way to handle this is to break the experiment into smaller experiments (i.e. assignment procedures) and express them as chained tasks. This is like the qualifications mechanism, only broader and requiring that we have more complete information about respondents.

Note that we do not list longitudinal tasks above. A fantastic extension of a crowdsourcing system would be to support panels, for time-varying tasks.

## System Components
AMT already has some of the major components we list here. What follows is a discussion of how we might design a crowdsourcing system, if the objective for the system designers were principled analyses, rather than profit.

### Scheduling
When scheduling tasks, we need some way of differentiating _temporal scale_. Some questions or tasks are not always relevant. Some questions have a particular lifetime. Other questions are important over longer periods of time (this brings us back to the longitudinal task mentioned above).

We already know of some challenges with scheduling imposed by systems such as AMT:

* **UI Design** has a huge impact on what users see. A good system balances the users needs with the requester needs. The users are loosely maximizing some utility function relating perhaps to their targeted rate of compensation (citation, someone? I've heard that people do this, but have not read the papers myself), or enjoyment. The requesters want the job done at the cheapest rate possible. We generally know the requirements of the requesters -- inferring the requirements of the users is typically a much harder task. Fundamentally, we are talking about a matching problem, though. At the moment, systems such as AMT just post the most recent jobs. This (amongst other shortcomings in the system) leads to degenerate behavior in the workers -- they participate in offline discussions about tasks, which compromises requesters' desire for no collusion and/or independence in responses. A _matching algorithm_ between users and workers would help. It need not be sophisticated -- just a random sample of the top $$n$$ posts for which the user is qualified. This is, in fact, a requirement for the above classes of tasks!

* **Centralized/shared data** would be useful for requesters. This would reduce repetition across tasks and corroborate results, leading to more **more robust results via replication**. An easy way to accomplish this would be to embed the system in an existing social media system, where users explicitly provide significant information about themselves, and where they reveal information about themselves implicitly.

* **Searching for work**: It is not clear to me whether users ought to be allowed to explicitly search for work. However, a search mechanism might also alert us to when collusion occurs.

* **Payment** Aside from the obvious technical/legal challenges of paying workers, we also want the payment system to not interfere with the revelation of users' utility functions. In particular, we do not want the payment system to mask user behavior. There was some work a few years ago that argued that paying people more did not result in better work, and I've written before about how I think that this is an artifact of the overly discretized way AMT is set up. Since then, I've seen: (1) Google Consumer Surveys, which uses surveys as a paywall, (2) some other Google Survey App that asks me marketing questions on my Android phone and gives me Play Credits, which I use to buy overpriced e-versions of comedienne's memiors, (3) Quizz, another Google system that uses hypochondriacs' specialized knowledge of WebMD to answer health questions, but doesn't pay them and (4) some Facebook survey system that I didn't actually get to play around with when I was there and have never been exposed to, probably because I spend so little time on Facebook.

* **Rating systems for requesters**: Clearly the biggest problem with AMT has been that there is no recourse for the rejection of work and lemon requesters are everywhere. This has lead to people using Turkopticon and other fora to discuss HITs. The lack of an anonymous system and the lack of an anonymous rating system for specific tasks pushes people to discuss tasks online, which undermines our no-collusion and independence requirements.

* **Redefining units**: For all of the discussion about the importance of no-collusion and independence, there may be cases where we want interaction between units. Of course, in these cases, the unit would be the interacting pair. Currently AMT does not allow this kind of interaction, and I am unsure if any other systems have these built in.


What's really provocative to me is that this is another extension of the human-in-the-loop way of looking at programming languages and systems. Of course here at UMass we've seen Lori Clarke et al. use process languages to formalize human-in-the-loop processes in the medical domain. What differentiates a process language from a programming language? The process language describes what should happen -- it's a fine model to use for the system as-in, in dynamic analyses. The real benefit to using programming languages is in the static analyses. There's always some meta-process or pipeline for a particular task. Embedding assumptions into programming languages is a way to push as much of the decision-making as possible as early in the pipeline as possible. It allows us to "see into the future" and constrain our actions so that we only do things that are sound with regard to the specification.
