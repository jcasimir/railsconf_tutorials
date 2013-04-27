---
layout: page
title: Introduction
section: TDD Tools, Techniques, and Discipline
sidebar: false
---

Welcome to our two-part TDD workshop. We will be covering how to add features to
a Rails application in a Test-Driven approach.

Our approach will be to introduce a topic, speak about it for a few minutes,
then live-TDD a feature.

## What TDD is and why do it

Test-Driven Development (TDD) is the practice of writing a test for your
application before writing code. We will go through many examples to give you
a sense of how it's done.

TDD is a complete shift in thinking and requires discipline to develop the
habit, however the resulting software is much better.

Some advantages of writing software this way:
- You set the expected outcome first, then write only enough code to make the
  test pass. In this way, extraneous code is limited.
- Forces you to consider the purpose of the code at the outset, which can narrow
  specificity
- Reduced bugs and rework
- A solid test suite tells you if a change or refactoring has broken previously
  working code
- Clearly written tests serve as a form of documentation

## Red-Green-Refactor

The practical approach to TDD is to write a failing test first, then write just
enough code to make that test pass, and finally refactor your code. This cycle
is known as "Red-Green-Refactor" based on the test output colors (failing tests
are often output in red).

In the process of getting a test to pass, you may need another feature, which
means you will write another test and try to make that pass. In the case of
Behavior-Driven Development (BDD), we set high-level test expectations and allow
them to guide us to what lower-level code is necessary as well.

Lets get started with [an example of Outside-In Development][1]

[1]: {% page_url 02_integration_testing %}
