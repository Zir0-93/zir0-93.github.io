---
title: "How Striff checks a pull request against the docs in its own repository"
date: 2026-09-25 10:00:00
og_image: /images/striff-pr-path.png
tags: [code review, static analysis, software architecture, documentation, llm, system design]
toc: true
description: "How Striff turns the claims in a repository's own documents into rules and checks each pull request against them at both revisions, with a program deciding every verdict."
redirect_from:
  - /detecting-architectural-anomalies-gnn/
excerpt_separator: <!--more-->
---

When [Striff](https://striff.io) reviews a pull request, a language model reads your README, your architecture notes and your `AGENTS.md`, and it has no say in whether your code broke any of them. Its one job is to translate a sentence like "the domain layer must not depend on infrastructure" into a rule in a small formal language. A program checks that rule against the parsed code at the base of the pull request and again at its head, and the GitHub check run shows the verdict next to the sentence it came from. This post follows a pull request from the webhook to the last row of that check run, and explains the two rules that shaped most of the design along the way.

<!--more-->

## What lands on the pull request

The output is a GitHub check run with a summary, up to three top review items, a Documented Rules table (Rule, This PR, Source) and the architectural diff of the change as a diagram. Broken and restored rules always get a row. Rules that were already broken before the change get up to ten. A rule that still holds appears only when the change touched it.

The contract behind that table is lopsided on purpose: missing a rule is acceptable, and stating a wrong one is not. When the pipeline cannot tell, the check run says nothing and the API response and logs keep the full record. Most of the machinery below exists to make that contract hold.

## Two rules that outrank convenience

Striff never compiles your code. [clarpse](https://github.com/hadi-technology/clarpse) parses the source into a model of types, members and references, and [striff-lib](https://github.com/hadi-technology/striff-lib) turns the difference between two such models into a diagram. The price of skipping the build is that a source parser sees less than a compiler. It misses members a code generator adds, annotations it cannot resolve, and the string and class literals inside method bodies.

So the first rule is that absence from the parsed model is never evidence of absence in the code. Any code that concludes "X is not there" has to know whether the question was answerable, and say so when it was not.

The second rule is that "could not analyse" must never persist as "analysed, found nothing". From outside, both look like a quiet check, and a quiet check that means "I did not look" teaches a team to trust silence nobody earned. Every path that records a result has to answer what it writes when the work did not happen.

## From webhook to worker

![The path of a pull request through Striff](/images/striff-pr-path.svg){: .light-border }
*A webhook becomes a queued job. The worker is released once the structural result is posted, and the review completes the check run later.*

A GitHub App webhook is verified, queued and answered with a 202, and the check run is posted as queued off the request thread. The queue is one Redis queue with two lanes. Submissions from the browser extension, where a person is watching a spinner, go in the interactive lane, which is always drained first.

Each pod runs one worker and admits one analysis at a time. An analysis holds both parsed revisions in memory, and two large ones side by side can exhaust the heap and kill everything in flight on the pod. A cluster-wide semaphore would not help, because the heap belongs to the pod. The permit is taken before anything is downloaded, and waiters are ordered GitHub App first, then the extension, then the public API. Priority never preempts a running analysis.

A waiter gets 240 seconds. An App check that does not get a slot goes back on the queue with a delay, and its check run stays in progress, saying "Waiting for analysis capacity" and when Striff will stop trying. Failing it would have been simpler, but a refusal is a case of "could not analyse", and a completed check run cannot be reopened.

Inside the slot an analysis has 600 seconds. At the deadline a watchdog interrupts the thread. If the thread ignores the interrupt for another 30 seconds, the pod records the overrun and exits, and that revision is refused on the same build from then on.

When the analysis succeeds, the operation is saved, the diagram is posted into the still-running check run, and the run is handed to the review. The worker moves on. The review keeps only what it reads (the document catalogue, the relations at both revisions, the head source text), so the parsed models can be collected as soon as the analysis returns. Whoever first sees the review end claims the run in the database before posting, so exactly one final result reaches GitHub. If the review fails, times out or dies with its pod, the run shows the structural result alone, with no rule rows and nothing said about the check. A sweep finds reviews whose heartbeat has lapsed and completes their runs the same way.

## Which two revisions, and how much of them

The base is the pull request's merge base, which is often older than the tip of the target branch. GitHub's list of changed files is a three-dot diff, so applying it to the branch tip would compare against a merge nobody performed. Striff downloads the base archive once and builds the head by applying the changed files to a copy.

![What Striff parses for a change](/images/striff-parse-scope.svg){: .light-border }
*The changed files are parsed in full, what they reference is modelled beside them, and the rest of the revision is there so names resolve.*

The changed files are analysed in full. The repository files they reference are modelled beside them as boundary components, and the whole revision sits on disk so references resolve. Both simpler options failed first. A whole-repository parse was all or nothing and ran out of memory on large repositories. Parsing a copy cut down to the changed files could not tell a reference to an unloaded repository type from a reference to a library. With the revision on disk, the far end of every reference is known and the cost follows the size of the change.

The diagram draws the changed components and whatever sits on a relation the change added or deleted. The cost is that an unchanged caller is invisible: its relation to the changed code is identical at both revisions, and the analysis follows references outwards only. The diagram says what the change did and nothing about who else will feel it.

Boundary components are modelled only as deeply as the change reaches them, so the first rule applies. A rule that finds nothing among them has not been checked and cannot report that it holds, while a violation their present edges witness still stands.

Documents get a second pass. After the first parse, Striff chooses the documents this change bears on and parses the files declaring the types they talk about in full, so rules about those types are answered from complete dependencies. A pull request that changes only documents is scoped to the files those documents name. If they name nothing the repository declares, the check says "Nothing to analyze" instead of parsing a whole repository the change never mentioned.

## Which documents get read

![How Striff turns documents into checked rules](/images/striff-doc-rules-pipeline.svg){: .light-border }
*Every stage can decline a document or a rule. Only the translation step uses a language model.*

The catalogue lists the documents as they stand at head. Agent instruction files count as architecture documents, since `AGENTS.md`, `CLAUDE.md`, `copilot-instructions.md` and Cursor rules all say how the code should be shaped. Agent working memory under directories like `.claude/` is excluded, with release notes, migration guides, tests, vendored code and build output. So is a document whose header says it is superseded or not started.

Relevance comes from what the change did: the types its files declare, the far ends of relations it added or deleted, the libraries it starts using and the modules it touches. A type the change merely keeps using does not count, because a document about it is about the repository in general. Documents are ranked by how many of the change's names they mention and at most 25 are read. Those the cap holds back are reported unread, because a limit on spend must never look like a repository with nothing to say.

A document must also name at least one type in the parsed model, and method listings are skipped. Then a cheap model gets one question: could a change to this repository's source make something in this document false? A document saying which class declares which member, or which module depends on which, passes. A changelog does not. A skip is stored as a skip, and a document that already has stored rules is never skipped, because its edit has to be judged against what it used to state.

## Sentences into rules

The translation model sees each document as numbered sentences, with the closed list of names that exist in the two parsed revisions and a predicate reference with worked examples. Table rows and the edges of Mermaid or C4 diagrams become synthetic sentences so they can be cited. It returns JSON rules, each citing a sentence number and saying whether the document asserts the configuration or forbids it. A rule is a flat conjunction of predicates. "The domain layer must not depend on infrastructure" comes out roughly as:

```
in(a, "domain"), in(b, "infrastructure"), refs(a, b, "_")      expect: false
```

"The scheduler drives the engine" has the same shape with `expect: true`. The predicates cover where a component lives, what it declares, what it references or reaches transitively, and what the change added or deleted. Roles are normalised across languages, so a rule says `type_role(c, CONTRACT)` and never "interface". Where a language cannot express a role at all, as with interfaces in Python, the question is unanswerable for that component and never counts as false.

Everything the model returns is a candidate, and eighteen gates stand before evaluation. A rule must cite a sentence that was actually sent, and that sentence must not sit under a heading about planned work, known limitations or superseded content. Every name must exist in the parsed model at base or head. The exception is a name the parse could never have held, like a type generated from a schema or one under a test root: that rule is kept and answers "unsupported" with the reason, because "we cannot check this" tells a reader more than silence. Placeholders like `MyService` are refused, as is a rule that fails the type checker, contradicts itself or only says a component exists.

The gates run again on every cache read. The rule cache keeps one entry per document version, keyed by path and content hash, so a new gate reaches rules extracted months ago without another extraction. When the way documents are read changes, such as how sentences are numbered, a schema version is bumped and every older entry becomes a miss. One review gets 300 seconds for screening and extraction. A document the budget does not reach is reported unread and never cached, so a later review picks it up.

## Six outcomes at two revisions

Each surviving rule is evaluated by a backtracking join over the relations read from the parsed model at base and at head. No model is involved here. The pair of truth values picks the outcome:

| Outcome | Meaning |
|---|---|
| Holds | The change does not break the rule. Nothing is claimed about the rest of the code. |
| Violated | True at base and false at head, or a forbidden edge the change introduced. |
| Pre-existing | Broken at both revisions, with a named witness, and not by this change. |
| Restored | The document and the code disagreed at base, and the change closed the gap. |
| Unresolvable | The scope was empty or a name resolved to nothing. |
| Unsupported | The question cannot be answered against this model, and the reason is named. |

The last two carry the first rule. An assertion false at both revisions counts as pre-existing only for a declaration whose owner resolves to a source file. Any other false-at-both is unsupported, because a missing edge looks exactly like a parser that could not see it. A rule over a relation the model does not populate is unsupported too, since running it would report a clean result for a question never asked. Where the model lacks a member, the evaluator reads the head source text, and answers "read and not named" only when that evidence is complete.

Every guard in the evaluator can remove a witness or turn a verdict into unsupported, and none can create a verdict. An accusation that names no component becomes unsupported. And if the pull request itself wrote the cited sentence, a restored rule counts as a plain hold, because editing the document to match the code earns no credit.

## Could not analyse is never found nothing

The second rule shows up in small decisions everywhere. An operation record is written only after an analysis succeeds, so a failure never leaves an empty result that could be served as clean. The count of documents read stays empty unless the documented-rule check finished, which lets "0 of 14 documents" mean that none bore on the change. A review nobody can be shown any more, because the pull request moved on or the App was suspended, is recorded as failed with its reason and its verdicts are discarded. Caches make a repeated request cheap and are never the authority for a result, and a review is never rebuilt from a cache.

Only a computed finding can create an item a reviewer sees, and the model is never shown the verdicts or findings. What the model reads and what reaches a reader are decided by different code.

One producer sits beside the rule pipeline and needs no change to break anything: does a document name a type the repository does not have at all? It reads the whole revision's file tree and the repository's history, and reports one finding per stale page. I wrote about one case it found in [a post on a Copilot instructions file that names a class Copilot deleted]({% post_url 2026-09-26-copilot-instructions-deleted-class %}). The product side of the pipeline is on the [Striff blog](https://striff.io/blog/design-docs-are-enforceable-now).

## What this costs, and what it misses

The design trades recall for precision. A document that names none of your types is never read. A rule whose subject the parser cannot see comes back unsupported and stays out of the check run, so a quiet Documented Rules section says nothing about whether your docs are accurate. The 25-document cap and the time budget mean a cold repository with many documents is read over several reviews. The screening model can wave a relevant document away, which only ever shows up as a miss. The translation model can misread a sentence, which is why every rule shows the sentence it came from.

The service itself is a Spring Boot application on Kubernetes, with MongoDB as the system of record and Redis optional for queues and caches. GitHub Actions builds the image and Argo CD applies the manifests, much like the [MLOps blueprint]({% post_url 2024-01-09-mlops-blueprint %}) I described earlier. Public repositories are analysed for free.

---

## Code referenced in this post

<div class="post-badges">
<a href="https://github.com/hadi-technology/striff-lib"><img src="https://img.shields.io/badge/Diff%20and%20Diagrams-striff--lib-blueviolet?logo=github" alt="striff-lib"></a>
<a href="https://github.com/hadi-technology/clarpse"><img src="https://img.shields.io/badge/Source%20Parsing-clarpse-6a0dad?logo=github" alt="clarpse"></a>
</div>

striff-lib is on Maven as `io.github.hadi-technology:striff-lib`.

