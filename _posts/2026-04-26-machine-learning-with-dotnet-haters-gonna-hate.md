---
layout: post
title: "Machine Learning with .NET - Haters Gonna Hate"
date: 2026-04-26 09:00:00 +0400
categories: dotnet ai machine-learning
published: false
excerpt: "A practical take on using .NET for machine learning systems, where it fits well, and why the ecosystem deserves more credit than it usually gets."
---

This post is based on my talk at [Dotnet Georgia](https://dotnet.ge/Events/Detail/65c0690a-4d43-46bd-b5de-f8d317663bb2), which was not recorded unfortunately.

I wanted to turn that talk into a written version focused on the practical side: what machine learning is, where it sits in the AI landscape, and why using it from .NET is not as strange as some people like to pretend.

Every time someone says "machine learning with .NET", part of the room starts smiling like the joke already wrote itself.

Python owns the ML conversation. That is not controversial. The ecosystem is huge, the research community publishes there first, and most examples, notebooks, and libraries assume Python by default.

But that does not mean .NET is irrelevant.

It means you need to be honest about the job you are trying to do.

## AI Landscape Overview

Before talking about machine learning, it helps to separate the different meanings people put behind the word "AI".

AI is a very broad term. It can mean anything from a recommendation system to the kind of human-level intelligence people imagine in science fiction.

In practice, people usually talk about three broad categories.

## Narrow AI

Narrow AI is the AI we actually have today.

This includes systems like ChatGPT, image recognition, spam detection, recommendation engines, fraud detection, and document classification.

These systems can be very impressive, but they are still built for limited contexts. They are trained or designed to do specific kinds of work.

A recommendation system does not suddenly become a good legal assistant. An image classifier does not become a project manager. A language model can look general because language is a general interface, but even there, the system still has limits.

That distinction matters because most real engineering work today is not about building human-level intelligence. It is about using narrow AI to solve specific business problems.

## Strong AI

Strong AI, or AGI, is still hypothetical.

The idea is an AI system that can learn, reason, adapt, and solve problems across domains like a human can. It would not be limited to one task or one narrow context.

It could move from one area to another without being rebuilt from scratch for every problem.

That does not exist yet.

There is a lot of progress in AI, but production software teams should be careful not to confuse current systems with AGI. The tools we have are useful, but they are not magic.

## Super AI

Super AI is even further away.

This is the idea of AI that is more capable than humans in essentially every area: reasoning, creativity, decision making, emotional intelligence, scientific discovery, and more.

It is the category people usually talk about when they discuss AI reshaping society at a global level.

Maybe that will happen one day. Maybe it will not happen the way people imagine it.

For this post, it is not the important part.

The important part is this: the AI we build with today is narrow AI. Machine learning is one of the main ways we build it.

## What Is Machine Learning

Machine learning is a subset of AI.

Instead of explicitly programming every rule, we give the system data and let it learn patterns from that data. Then the trained model can make predictions or decisions on new input.

That definition sounds simple, but it changes how we think about software.

In a traditional program, we usually write the rules ourselves:

- if this value is above a threshold, do this
- if this text matches a pattern, classify it that way
- if this user has this role, allow this action

With machine learning, the rule is not always written directly by a developer. The model learns from examples.

That makes it useful for problems where the rules are hard to write by hand, but examples are available.

The main types are straightforward.

Supervised learning means the model learns from labeled data. For example, historical support tickets where each ticket already has the correct category.

Unsupervised learning means the model tries to find patterns in data without labels. For example, grouping customers by behavior when you do not already know the groups.

Semi-supervised learning uses a small amount of labeled data together with a larger amount of unlabeled data. This is useful when labels are expensive, but raw data is available.

Reinforcement learning is different. The system learns by interacting with an environment and receiving feedback. It tries actions, gets rewards or penalties, and improves over time.

Most business applications do not need the most exotic version of machine learning.

They need something practical:

- predict a number
- classify an item
- detect an anomaly
- rank results
- recommend something
- extract useful information

That is where ML.NET becomes interesting for .NET developers.

## Meet ML.NET

ML.NET is a machine learning framework built by Microsoft for the .NET ecosystem.

It is open-source, cross-platform, and designed for developers who want to build machine learning features using C# or F#.

It works on Windows, Linux, and macOS. It can be used from console apps, workers, Web APIs, desktop apps, Blazor apps, MAUI apps, and other .NET workloads.

That matters because the model is usually only one part of the system.

The rest of the system still needs the normal things:

- APIs
- validation
- storage
- authentication
- observability
- background jobs
- deployment
- support after it goes live

If your application is already in .NET, being able to keep the ML workflow close to the existing codebase is valuable.

ML.NET can be used for scenarios like:

- classification
- regression
- recommendations
- clustering
- anomaly detection
- text analysis and NLP
- ranking
- image classification through integration with TensorFlow or ONNX

It is not trying to replace every Python tool.

That is the wrong comparison.

The useful question is whether it can help a .NET team build practical predictive features without moving the whole system into another ecosystem.

For many scenarios, the answer is yes.

## Why Machine Learning in .NET?

The strongest reason is simple: you are already there.

If the product, team, deployment pipeline, monitoring, and operational knowledge are built around .NET, then adding a separate Python service has a cost.

Sometimes that cost is worth it.

If the work is research-heavy, Python is usually the right default. The ecosystem is bigger, the libraries are stronger for deep learning, and the examples are everywhere.

But many companies are not doing ML research.

They are trying to put model-backed behavior into existing products:

- classify incoming data
- rank candidates
- detect anomalies
- extract information from documents
- recommend the next action
- predict a business metric

That kind of work is not only about training a model.

It is also about integrating the model into the application safely.

The pros are clear:

- you can stay in the .NET ecosystem
- ML.NET is production-ready and supported by Microsoft
- you can build predictive features using C# and .NET
- it integrates naturally into APIs, workers, desktop apps, and UI applications
- you can deploy it with the same patterns your team already uses

The cons are also real:

- the community is smaller than Python's ML ecosystem
- there are fewer tutorials, examples, and Stack Overflow answers
- deep learning support is limited compared to Python-first frameworks
- for advanced scenarios, you often depend on TensorFlow, ONNX, or a separately trained model

That is the tradeoff.

For some teams, Python is the right answer. For some teams, .NET is enough. For many teams, the best architecture is both.

Train where it makes sense. Serve where it makes sense.

## How ML.NET Works

ML.NET usually follows a pipeline.

That pipeline is one of the reasons it feels familiar once you understand the basic pieces. You are not just throwing data into a black box. You are describing a sequence of steps.

The common flow looks like this:

1. Load data
2. Transform data
3. Choose an algorithm
4. Train the model
5. Evaluate the model
6. Make predictions

That is the mental model.

## 1. Load Data

First, you need data.

That data can come from CSV files, JSON files, databases, APIs, or in-memory collections.

For demos, CSV is usually the easiest place to start. In a real application, the data often comes from a database or from events your system already collects.

This is also the first place where teams usually underestimate the work.

A model is only as useful as the data behind it. If the data is inconsistent, biased, incomplete, or not representative of production, the model will reflect that.

## 2. Transform Data

Raw data is rarely ready for training.

You usually need to clean it and convert it into a shape the algorithm can use.

In ML.NET, this is done with transformers. Typical transformations include:

- normalizing numeric values
- converting text into numeric features
- one-hot encoding categorical values
- combining multiple input columns into a feature vector

This step is not glamorous, but it is important.

A lot of model quality comes from how well you prepare the data before training.

## 3. Choose Algorithm

After the data is ready, you choose a trainer.

The trainer depends on the kind of problem you are solving.

For example:

- `FastTree` can be used for regression scenarios
- matrix factorization can be used for recommendations
- SDCA logistic regression can be used for binary classification

You do not need to memorize every algorithm before building anything useful.

You need to understand the problem type first. Are you predicting a number? Choosing between categories? Finding similar users? Detecting unusual behavior?

Once that is clear, the algorithm choice becomes much easier.

## 4. Train the Model

Training means fitting the model to your data.

The model looks at the examples, learns patterns, and produces something that can later be used for prediction.

This is the part people usually imagine when they hear machine learning, but in practice it is only one step in the pipeline.

Loading, cleaning, transforming, evaluating, and deploying matter just as much.

## 5. Evaluate the Model

After training, you need to measure the model.

You usually do that with test data that was not used during training.

The exact metric depends on the scenario:

- accuracy for many classification problems
- RMSE for regression problems
- precision and recall when false positives and false negatives have different costs

This step is where you find out whether the model is actually useful or only looked good during training.

Skipping evaluation is how you ship a confident bug.

## 6. Make Predictions

Once the model is trained and evaluated, you can use it for predictions.

In a production system, that might happen inside a Web API, a background worker, a desktop app, or a scheduled job.

The important part is that prediction becomes just another part of the application flow.

Input comes in. The model produces an output. The application decides what to do with that output.

That decision still needs engineering judgment.

The model can help classify, rank, score, or recommend. It should not automatically become the owner of the whole business process.

## Custom Models, Pre-Trained Models, and AutoML

ML.NET supports custom models trained on your own datasets.

It also supports using pre-trained models through integrations like ONNX. That means you can train a model in another ecosystem, such as PyTorch or TensorFlow, export it to ONNX, and consume it from .NET.

This is a very practical architecture.

You can use Python where experimentation and training are strongest, then use .NET where production service ownership is strongest.

ML.NET also supports AutoML.

AutoML helps search for a good model and parameters automatically. That is useful when you are new to machine learning or when you want a baseline quickly.

It does not remove the need to understand the data or evaluate the result, but it can make the first step much easier.

## Where .NET Fits

.NET will not replace Python as the default language of machine learning research.

It does not need to.

.NET only needs to be a strong option for building reliable ML-backed systems. And for many companies, that is the actual problem they have.

If your team already runs production services in .NET, there is nothing embarrassing about using .NET for the AI parts too.

Use Python where Python is better. Use .NET where .NET is better.

Then ship the system.

## Further reading

- [ML.NET overview](https://learn.microsoft.com/dotnet/machine-learning/overview)
- [What is ML.NET?](https://dotnet.microsoft.com/learn/ml-dotnet/what-is-mldotnet)
- [ML.NET and ONNX in .NET](https://learn.microsoft.com/azure/machine-learning/how-to-use-automl-onnx-model-dotnet)
