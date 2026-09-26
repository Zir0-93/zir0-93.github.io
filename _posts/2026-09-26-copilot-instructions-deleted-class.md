---
title: "The MCP C# SDK's Copilot instructions tell agents to use a class Copilot deleted"
date: 2026-09-26 09:00:00
og_image: /images/og-copilot-instructions.png
tags: [coding agents, documentation, architecture, software design]
toc: true
description: "Line 258 of the official C# SDK's copilot-instructions.md says to use McpServerFactory. Copilot wrote that line in October 2025 and deleted the class in December. Ten months on, every agent that opens the repository is still told to use it. How that happens, why a normal pipeline never notices, and what to do about it."
excerpt_separator: <!--more-->
---

Every coding agent opens the same file first. Before it writes anything it reads the repository's instruction file, whichever of `CLAUDE.md`, `AGENTS.md` or `.github/copilot-instructions.md` the project keeps, and takes what that file says about the code as a starting point. Here is line 258 of the one in the [official C# SDK for the Model Context Protocol](https://github.com/modelcontextprotocol/csharp-sdk), the protocol whose whole job is to give models accurate context:

<div class="fig">
<div class="ghfile">
<div class="ghfile-bar"><span class="ghfile-crumbs">csharp-sdk <i>/</i> .github <i>/</i> <b>copilot-instructions.md</b></span><span class="ghfile-tabs"><span class="is-on">Preview</span><span>Code</span><span>Blame</span></span></div>
<div class="ghfile-body">
<p class="ghfile-h">Server Implementation</p>
<ul>
<li><span class="ghfile-ln">256</span><span class="ghfile-text">Server primitives (tools, prompts, resources) are discovered via reflection using attributes</span></li>
<li><span class="ghfile-ln">257</span><span class="ghfile-text">Support both attribute-based registration (<code>WithTools&lt;T&gt;()</code>) and instance-based (<code>WithTools(target)</code>)</span></li>
<li class="is-hl"><span class="ghfile-ln">258</span><span class="ghfile-text">Use <mark>McpServerFactory</mark> to create server instances with configured options<br /><span class="ghfile-callout"><b>Not in the code.</b> Deleted 2 Dec 2025 in <a href="https://github.com/modelcontextprotocol/csharp-sdk/pull/985">#985</a>.</span></span></li>
</ul>
</div>
</div>
<p class="fig-caption"><a href="https://github.com/modelcontextprotocol/csharp-sdk/blob/c40ee044fd415c70da5176c749cb5ef02f2b59f6/.github/copilot-instructions.md#L258">The line, at the commit I checked</a>. This file exists to brief a coding agent before it writes anything.</p>
</div>

`McpServerFactory` does not exist. Search the repository for the name and you get one result: this line, in the file every agent reads first. The class was marked obsolete in September 2025 with a note to use `McpServer.Create` instead, and deleted in December.

<!--more-->

<div class="callout">Copilot wrote that instruction. Seven weeks later, Copilot deleted the class it names. Ten months on, the line is still there.</div>

<div class="stats">
<div class="stat"><div class="stat-value">1</div><div class="stat-label">search result for <code>McpServerFactory</code> in the whole repository: this line</div></div>
<div class="stat stat--amber"><div class="stat-value">7 wk</div><div class="stat-label">between Copilot writing the instruction and Copilot deleting the class</div></div>
<div class="stat stat--danger"><div class="stat-value">10 mo</div><div class="stat-label">the line has stood since the deletion, through four edits of the file</div></div>
</div>

To be precise about the cost: an agent that takes line 258 at face value writes `McpServerFactory`, watches the build fail, and goes looking for what it should have done. A more careful agent greps for the name first, finds nothing, and does the same search a few turns earlier. Either way the damage is a few wasted turns, which is cheap. The reason I wrote this up is what it says about every other sentence in that file, and in yours.

## How the line got there and stayed

Nobody involved would have caught this, because nothing they did required them to:

<div class="fig">
<p class="fig-title">One line, five commits</p>
<ol class="timeline">
<li><span class="tl-date">16 Sep 2025</span><span class="tl-body"><code>McpServerFactory</code> is marked <code>[Obsolete]</code>: <em>"Use McpServer.Create instead. This member will be removed in a subsequent release."</em> <a href="https://github.com/modelcontextprotocol/csharp-sdk/commit/38b4a269">38b4a26</a></span></li>
<li><span class="tl-date">13 Oct 2025</span><span class="tl-body">Copilot opens <a href="https://github.com/modelcontextprotocol/csharp-sdk/pull/858">#858</a>, "Set up Copilot instructions for repository", a long and mostly accurate briefing. A maintainer reviews and merges it. The instruction to use <code>McpServerFactory</code> is line 222.</span></li>
<li class="is-del"><span class="tl-date">2 Dec 2025</span><span class="tl-body"><a href="https://github.com/modelcontextprotocol/csharp-sdk/pull/985">#985</a>, "Remove obsolete APIs from codebase", deletes <code>McpServerFactory.cs</code>. Authored by Copilot, co-authored by three of the project's developers. The instructions file is not in the diff, so nobody reviewing the removal opened it.</span></li>
<li><span class="tl-date">Apr to Aug 2026</span><span class="tl-body">Four more commits edit the instructions file. None touches the line, which drifts down to 258.</span></li>
<li class="is-now"><span class="tl-date">Today</span><span class="tl-body">Line 258 is still there.</span></li>
</ol>
</div>

The reviewer of the instructions read a long document that was almost entirely right. The reviewer of the removal read a diff, and the document was not in it. That is the whole mechanism. A document and the code it describes live in different files, and a pull request only ever shows you one of them.

## The agent repeats the wrong sentence

Stale names do not just sit there. In April 2026 a commit authored as "Architecture Bot" [added an Architecture section](https://github.com/yegor256/cactoos/commit/18a0c8ad8d8b64d6bbb62fa36e668f967afda2ac) to the README of [yegor256/cactoos](https://github.com/yegor256/cactoos). The commit message links a Claude Code session. The new section says:

> Explicit caching requires opting in with `Sticky` or `StickyList`.

`StickyList` was [renamed away in December 2018](https://github.com/yegor256/cactoos/commit/be02845), and the list class it became was [removed in 2020](https://github.com/yegor256/cactoos/commit/eed03c1fc05217eeda0564999cae2c87709b1e13). The old name survived in a comparison table further up the same README. An agent asked to describe the architecture read that table, and the name came out the other side as a confident new instruction with a fresh timestamp. The next agent to open that README finds two sentences recommending a class that has been gone for seven years, and the newer one looks authoritative.

<div class="callout callout--amber">A wrong sentence in a doc used to cost one engineer an afternoon of confusion. Now every agent run reads it, acts on it, and sometimes copies it into the next document.</div>

## Why your pipeline does not catch this

Design docs used to have one reader, a person, who noticed when a sentence had stopped being true. Now the document is an input to code generation. The practice has a name, spec-driven development, and a toolchain: GitHub's Spec Kit, AWS's Kiro, Tessl, and the `AGENTS.md` and `CLAUDE.md` files that brief an agent before it touches anything. Birgitta Böckeler's [survey of those three tools](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html) sorts the practice into three levels: spec-first, where the spec drives one task; spec-anchored, where it is kept afterwards and the feature keeps evolving through it; and spec-as-source, where a human edits only the spec and never the code. Everything past the first level depends on the spec staying true after the task ends, and that is the step the tooling has not caught up with.

Everything else your code is built from has a check: source has a compiler, tests have a runner, types have a checker, manifests have a resolver. Prose has some checks too, and they are worth naming so nobody thinks I am pretending otherwise. Rustdoc warns on a broken intra-doc link. Doctests run the examples. A link checker catches a dead URL. None of them reads a sentence. "Use `McpServerFactory`" is not a link, not an example and not a URL, so it passes all of them.

The obvious answer is to grep. `git grep -w McpServerFactory -- '*.md'` takes a second, and if every removal PR ran it, this post would have nothing to show you. I would genuinely like everyone to do that, and it is the first item in the list at the end. It also only catches the version of the problem you can see. The expensive version has no name in it.

<div class="fig">
<p class="fig-title">Two ways an agent strays from the docs</p>
<div class="outcomes">
<div class="outcome outcome--amber"><p class="outcome-name">It follows a sentence that is no longer true</p><p class="outcome-desc"><em>"Use <code>McpServerFactory</code>."</em> The build breaks, the agent recovers, you pay in turns and tokens. If the sentence describes a pattern rather than a class, the build does not break, and the agent reproduces a design the team retired.</p></div>
<div class="outcome outcome--danger"><p class="outcome-name">It ignores a sentence that is still true</p><p class="outcome-desc"><em>"Controllers go through the service layer." "<code>core</code> never imports from <code>plugins</code>."</em> The agent takes a shortcut. The code compiles, the tests pass, the diff looks fine, and the document that forbade it is not in the diff. Merged.</p></div>
</div>
</div>

The second case is where architecture goes: a hundred small changes, each of which compiled, each reviewed as a diff, each contradicting a sentence nobody had open. A year later the layering the team agreed on describes a system that no longer exists. Grep cannot find that, because there is no token to search for. Checking "`core` never imports from `plugins`" means turning the sentence into a question about the dependency graph and asking it at both ends of the pull request.

## It is not rare

The MCP line is the one I chose to lead with because the irony is hard to beat, but it was not hard to find. In the same sweep of public repositories, [DolphinScheduler](https://github.com/apache/dolphinscheduler/blob/dev/docs/docs/en/contribute/backend/spi/registry.md?plain=1#L20)'s contributor guide sends new contributors to implement an interface that was removed in a package split, [Spring AI](https://github.com/spring-projects/spring-ai/blob/main/spring-ai-docs/src/main/antora/modules/ROOT/pages/api/audio/speech/openai-speech.adoc)'s speech reference walks through a class the project replaced five months ago, and [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet/blob/master/docs/articles/configs/exporters.md?plain=1#L101) documents four properties of an interface that no longer exists.

<div class="callout callout--green">These are well-maintained projects with careful reviewers. That is the point: the review process is fine, it just never puts the sentence and the code in front of the same person.</div>

## What to do about it

- Treat your agent instruction files as code. They are an input to your codebase now. When you delete or rename a type, `git grep -w OldName -- '*.md'` takes a second, and IDE rename refactorings skip markdown.
- Keep them short, and prefer rules to names. "Controllers never call repositories directly" stays true across a hundred refactors. A class name is a claim that can go stale on the next one.
- Have the docs checked on the pull request, where the change that contradicts them is being reviewed, by something that reads the sentence and the code together.

That last one is what I build, which is how I found these. [Striff](https://striff.io) is a GitHub App, free on public repositories, that parses both revisions of a pull request, reads the documents already in the repository, turns each sentence that makes a claim about the code into a rule, and checks it at the base and the head. A name the repository does not have, like line 258, is reported against the page with the commit that removed the type. A rule the change broke is reported against the change. There is nothing to write and nothing to configure, because the rules are the ones your team already wrote down and your agents are already reading. The sweep these examples came from, with its numbers and a worked example from sentence to verdict, is in [a post on the Striff blog](https://striff.io/blog/design-docs-are-enforceable-now).

I opened fixes for both lines before publishing this: [modelcontextprotocol/csharp-sdk#1892](https://github.com/modelcontextprotocol/csharp-sdk/pull/1892) (with [issue #1893](https://github.com/modelcontextprotocol/csharp-sdk/issues/1893)) and [yegor256/cactoos#1959](https://github.com/yegor256/cactoos/pull/1959).

*Method notes.* Every repository named above was checked by hand against its default branch in September 2026, and the primary links go to commit permalinks so they survive the fix. A documented type counts as absent only on an exact-token match against the whole revision, with the repository's history checked to tell a moved type from a removed one. Code is parsed from source without compiling it; where that leaves a claim unanswerable, the check reports nothing rather than a pass or a failure.
