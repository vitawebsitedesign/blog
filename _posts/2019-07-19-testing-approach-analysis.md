---
title: Hollistic cost/benenfit analysis of testing approaches
layout: post
author: Michael Nguyen
---
Below is summary of common test approaches:

### Manual unit testing
Difficult to consistently obtain high test coverage

Difficult to obtain consistent test results among testers

High business cost to maintain test process & increase test scope.

Despite the downsides, sometimes regulatory bodies require manual testing (e.g.: medical devices), so software companies in this position dont have much choice if they want to pass audits.

### Automated unit testing
Easy to implement.

Easy to integrate into bamboo.

Low business cost to maintain unit tests & add new ones.

### Automated integration testing
Moderate to implement (its basically unit testing w/ extra scope).

Easy to integrate into bamboo.

Moderate business cost to maintain integration tests & add new ones.

### Automated end-to-end integration testing
Difficult to implement (very steep learning curve, slow start, extremely large scope).

Difficlt to integrate into bamboo.

High business cost to maintain, moderate cost to add new tests.

## Whats the point of automated end-to-end integration testing then?
The facts above mostly relate to cost.

Something i've left out is the benefit.

In the software world, the point of testing (manual and/or automated) is to prevent bugs hitting customers.

Specifically, detection before public release, & quick fix after detection.

## Detection
Although automated end-to-end testing is expensive to build & maintain, it has significant coverage.

This means better detection (compared to ad-hoc manual testing).

## Quick fix after detection
After the bug is detected, it needs to be fixed quickly.

Again, auto E2E is taken for a quick spin on the fix.

Compared to ad-hoc manual testing, the delivery time to customer is significantly shorter.

And shorter delivery times means a better edge for a competitive business.

## Bottom-up vs top-to-bottom
If you dont have automated E2E testing framework integrated into your CI/CD, you'll wanna start bottom-up.

This means starting at the "cheap" manual ad-hoc testing, & moving "up" to the more "expensive" testing techniques WHEN IT IS PREFERRED.

"When preferred" means to only advance forward if (1) time allows, or your lead has prioritised it above critical bugs/features (2) your business encounters one of the following issues that can be solved by auto E2E:
- The business needs a faster delivery time (e.g.: hotfixes, releasing new features to react to competition)
- Bugs are hitting customers, which has slithered under-the-radar of your current testing approaches
- Your business software scope is so large that writing non-E2E tests becomes too expensive & difficult to achieve high test coverage
