---
layout: single
title:  "Karlsruhe to D.C. ― a Dask story"
date:   2020-05-28 20:00:00 +0100
tags: technology python data-engineering Dask distributed
header:
  overlay_image: assets/images/dask_icon.svg
  overlay_filter: 0.2
  show_overlay_excerpt: false
author: Lucas Rademaker & Nefta Kanilmaz
author_profile: true
---
# Karlsruhe to D.C. ― a Dask story

Back in February 2020, we (Florian Jetter, Nefta Kanilmaz and Lucas Rademaker) travelled to the Washington D.C. metropolitan area to attend the first ever Dask developer conference. Our primary goal was to discuss Blue Yonder’s issues related to Dask with the attending developers and users. When we returned back to Germany, not only had we connected with many of the core developers in the Dask community, but we had also (almost finished) an implementation of a distributed semaphore in our bags. 

There is [a blog post](https://blog.dask.org/2020/04/28/dask-summit) giving a general overview and summary of the workshop talks. This blog post is a summary of _our_ experience of the workshop.. 

## Setting

The three-day long workshop included sessions with short talks, as well as time slots for working on Dask-related issues and discussions.
The talks covered a broad spectrum of topics including Dask-ML and the usage of Dask in general data analysis, Dask deployment and infrastructure, and many more. All talks focused on answering these key questions: 

- Who are Dask users and what are their use cases?
- What are current pain points and what needs to be fixed as soon as possible?
- What is on the Dask users' wish list?

In other words, this was an opportunity for presenters to officially “rant” [*] about Dask, with experts having the experience and knowledge to help in the same room.
This exchange between users and developers enabled immediate fixes of minor problems, identifying synergies between different projects, as well as shaping the roadmap for future development of Dask. We feel that this is a successful model for driving the Dask open-source community.

[*] quote Matthew Rocklin

## The Blue Yonder way of using Dask

We at Blue Yonder are still finishing our migration from a proprietary job scheduler to Dask.distributed. This is why when _we_ talk about _Dask_ in this post, we are mostly referring to _Dask.distributed_, as this is our main use case.

We quickly realized at the workshop that Blue Yonder’s use case is almost unique in the entire Dask community.

Florian Jetter opened proceedings at the conference with a presentation on the typical data flow for our machine learning products, our usage of Dask and Dask.distributed within this flow and where we were currently facing issues.
 
At Blue Yonder, we provide Machine Learning driven solutions to our customers mainly in retail. The data - which our customers provide through an API - is inserted into a relational database. The Machine Learning and prediction steps need a denormalized data format, which we provide in the form of Parquet datasets in the Azure Blob Store. The resulting predictions are written back to Parquet files and offered to customers through an API.
The Data Engineering team leverages the distributed scheduler for parallel task execution during the data transformation steps between database and Parquet datasets. Most parts of these pipelines use Dask bags or delayed objects for map/reduce operations. Dask dataframes are especially useful when we reshuffle datasets with an existing partitioning to a differently partitioned dataset. We use [Kartothek](https://github.com/JDASoftwareGroup/kartothek) for this purpose.

A lot of Dask users we met at the workshop also utilized Dask clusters for data-heavy calculations. However, the Blue Yonder use case remains special: it appears that most other users are working in an R&D-like environment and their computations are not running in comparable production environments. Most importantly, these users have no service level agreements with customers: SLAs, for example, for delivery times for predictions. From our interactions with users, it became apparent that Blue Yonder puts higher requirements on the stability and robustness of the distributed scheduler than any other users present at the conference.

## Issues we have encountered with Dask and potential improvements

During discussions with conference attendees we touched upon some topics that have been (or still are) big problems for us and encountered other community members with similar experiences. Below are some of the "rants" we'd like to share.


### Distributed stability
We migrated some of our largest data pipelines to Dask in the last months of 2019. For these pipelines, we started to observe significant instability during the computation of the Dask graph. We hadn't seen this issue in any of our previous pipelines running on distributed. Our team contributed several patches upstream in order to resolve these issues on our side. 
 
During the working sessions, a number of conversations related to the stability of the distributed scheduler took place.
One idea which came up, to increase the overall robustness of distributed, was _replication of task results_. The idea is as follows: if a worker which is holding a task result is shut down, replicating the task result will allow certain parts of the graph to not need to be re-computed.
As of the time of this writing, we have not had a chance to actively work on something like this yet, but it is something that we keep in mind.

### Performance and graph optimization
While monitoring the execution of one of our Dask production pipelines, we observed that the amount of memory consumed by the workers was significantly higher than one would expect in an ideal scenario where tasks are executed in a way which minimizes memory usage of the workers.
When investigating the Dask.optimization module, we saw that code to optimize memory usage was already there. In practice, however, the graphs are not executed in the optimal order with regards to memory usage because of other constraints during execution.

A Dask user at the workshop facing this issue told us that they worked around this by injecting dummy dependencies into Dask graphs. These dependencies acted as "choke-holds" for certain types of tasks, in order to improve the memory usage during execution.

This lack of optimization, also referred to as memory back-pressure, not only impacts memory usage but also increases resource consumption. We would greatly benefit from an implementation which addresses this issue.

## Looking back

### Results from working sessions
Lucas had the pleasure to collaborate with John Lee on writing down a benchmark to enable [work stealing for tasks with restrictions](https://github.com/Dask/distributed/pull/3069). He also got valuable input from Tom Augspurger on dask internals while debugging a Dask.dataframe bug involving categoricals.

However, arguably the most useful work we did was the implementation of a semaphore in Dask.distributed.
Our workflows depend heavily on accessing resources like production databases. Migrating them to the distributed scheduler was therefore not possible without first being able to rate-limit the access of computation cluster to such resources. For this, we needed a semaphore implementation.
Coincidentally, we also talked with other attendees of the workshop whom were interested in such a functionality.
This is something we did not forget; despite the jetlag, we left D.C. with a good chunk of the implementation necessary for a semaphore. This finally got [released](https://github.com/Dask/distributed/commit/2129b740c1e3f524e5ba40a0b6a77b239d4c1f94) with distributed 2.14.0.

### Interaction with the community
Being able to share our issues with the Dask community and discuss potential ways of improvement with expert users and core developers was extremely valuable. Additionally, this interaction gave us a wider perspective of the current status of Dask, the ecosystem around it and what we can expect from it in the future.

We were also reminded once again of the value of open-source software. 
There was a clear synergy in terms of intended functionality between [Kartothek](https://github.com/JDASoftwareGroup/kartothek) and other tools in the ecosystem such as Arrow/Parquet and Dask.dataframe I/O and partitioning.
Dask is driven by a wide community of skilled and passionate developers. 
By committing to this community-driven project, we get to collaboratively shape the best software out there.

### Final remarks

We are grateful to have had the opportunity to attend this workshop.
We'd like to thank the organizers for their hospitality and the success of this workshop.

See you at the Dask developer workshop 2021 ;).
