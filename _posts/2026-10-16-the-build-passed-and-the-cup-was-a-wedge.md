---
layout: post
date: 2026-10-16 20:00:00 +0530
tags:
  - agents
  - architecture
  - github-pages
  - lessons
---

# The Build Passed and the Cup Was a Wedge: Lessons from a Pivot I Have Not Shipped

On 15 October 2026 I asked for a Jekyll site that GitHub Pages could build from vanilla HTML, CSS, and JavaScript, with no third-party dependencies. Pages has no server and no database, so the site had to live in the repository and run in the browser. I created [prompt-marketplace](https://github.com/rajeshradhakrishnanmvk/prompt-marketplace) myself. The GitHub connection on the agent could not create repositories, and Cursor's access to the new repo had to be granted after the repo existed.

The product I asked for was a prompt marketplace with a feed in the style of LinkedIn or Facebook. A visitor could add a prompt, put it in a category, and filter the list. There is nowhere to post a form, so each prompt is a Markdown file in `_prompts`, and the way onto the site is a pull request. The Add Prompt form opens GitHub's new-file page with the filename and the body already filled in.

## The feed I merged, and did not want

A Cursor cloud agent built that in [pull request #1](https://github.com/rajeshradhakrishnanmvk/prompt-marketplace/pull/1). Ten seed prompts. A Liquid include that writes a JSON index. Filters and search kept in the URL query string, so a filtered view has a permalink. `jekyll build --safe` passed. The pull request merged into `main` the same afternoon, and GitHub Pages is serving that build from the project site.

On the morning of 16 October I still did not want the product. I had no answer for why a stranger would open the tab. I asked how a site inside these constraints gets traffic. The architect supervising the work proposed four directions: a before-and-after gallery, a daily challenge, a niche library, and a template playground. I then had that architect create Crew73, six agents under its supervision: Trend Scout, Growth Strategist, Product Designer, Builder, Content Creator, and QA Critic.

## Three rounds, then a square of paper

The first brainstorm used material we already had. A daily data-engineering puzzle, Schemadle. A Stephen King puzzle. A Lakehouse playground. A quiz. I said to drop data engineering and Stephen King and go niche. The second round came back India-focused: an RTO-code map game, a lost-unit converter, and ham radio exam practice. I asked them to think more. The third round produced global micro-niches and what the brief called "wow" toys. QA Critic's top pick was an origami paint-and-fold toy. I picked it. I also chose to keep this repository and replace its contents.

Product Designer wrote the spec and named it Foldly. The sheet is a CSS 3D hinge tree. The flaps are `clip-path` shapes. There is no fold solver. The folds are keyframes. Painting is Canvas 2D, and a stroke can sit on the front or the back of the square. A share URL runs the strokes through `CompressionStream` deflate and encodes them as base64url. PNG and WebM export re-renders that picture to a canvas and records it with `MediaRecorder`. Each model has one page for search and for print. Content Creator wrote the copy, renamed the fortune hat to a samurai helmet (kabuto, id `samurai-helmet`), and labelled estimates and unknowns as estimates and TBDs.

## What the draft actually contains

Builder opened [pull request #2](https://github.com/rajeshradhakrishnanmvk/prompt-marketplace/pull/2) as a draft. Six models: a paper cup, a cat face, a dog face, a tulip with a flat head and no stem, the samurai helmet, and a jumping frog lite that starts from a half square and whose materials mention scissors. The diff adds 6,123 lines and removes 2,661 across 60 files. `jekyll build` passed. `baseurl` stays `/prompt-marketplace`, so a merge would replace the marketplace Pages is already serving.

The pull request is still a draft. I have not merged it. Foldly is not what Pages is serving.

<img src="/assets/images/foldly-pivot-agent-flow.svg" alt="Flow from the merged prompt marketplace through three brainstorm rounds to the Foldly draft, with QA Critic's do-not-ship verdict and the human gates at idea, repository, and merge" title="Crew73 proposes under one architect. I decide the idea, the repository, and whether the draft merges. Pull request #2 has not merged." style="width:100%; height:auto;">

## What Playwright found

QA Critic tested the draft with Playwright on Chromium, WebKit, an iPhone 13, and a Pixel 7. The verdict was do not ship.

In WebKit the painted front faces never render. `clip-path` inside `preserve-3d`, together with `backface-visibility`, drops the paint. The builder had already noted that Safari was not run on the Linux VM that produced the draft, and had put `-webkit-clip-path` and `-webkit-backface-visibility` on the face leaves as a workaround. The WebKit run showed the paint still missing. A `display` rule overrides the `[hidden]` attribute, so pieces that should stay off the page stay on it. The layout overflows on the phones. Placeholder TBD copy is still in the page: who can open a share link and how long it lasts, who is behind the site, whether it is free, how accounts and privacy work, and what a classroom needs.

The fold geometry is the failure I would block a merge on. The first folds move only the front layer, so none of the six finished models is recognisable. The cup is a flat wedge. The cat and the helmet are triangles. The frog is hidden by fake "cut" strips. The draft already calls the steps stylised hinge approximations: the cup lays its tip on the pocket, the frog's collapse is a pre-split followed by a two-hinge zigzag, the cat's tuck is a partial unfold of the back tip, and the helmet's brim is two nested hinges. Those sentences describe the mechanism. The last frame still has to be checked against a photograph of the paper.

The same report has smaller gaps. There is no `hashchange` listener, so the page does not follow a shared URL as it changes. Contrast is short. There is no `og:image`. The print scale is off.

The fixes underway are specific. Both layers have to travel with the moving flaps. Every model needs an end state checked against a reference photo. WebKit has to show the paint. A model that cannot be made accurate gets dropped. Until that lands, the draft stays open.

## What I am keeping

Validate the idea before the build. Pull request #1 met the brief I wrote. The feed filters in the browser, the query string holds the view, and a contribution is a Markdown file because Pages cannot store one any other way. I merged it, and Pages is serving it. I still had no reason for a visit. The cheaper moment to say so was before the agent opened the pull request.

The constraint is part of the design. With no backend, the site has no accounts, no live write, and no feed except a rebuild. The marketplace spent its effort imitating a social network inside that box. Foldly puts the interaction in the browser: paint on a canvas, a compressed URL as the save file, and a Jekyll page per model. The same limit that made the marketplace awkward is a limit this toy can inhabit. Today it is a specification and an open draft.

An adversarial reader with a real browser sees what the author's own checks skip. The builder recorded a walkthrough and reported a passing Jekyll build. QA Critic opened Chromium and WebKit, plus two phone profiles, and both the painted fronts and the finished models failed. I want that pass on the next build, with a photograph of the folded model beside the last keyframe.

A passing build says the site compiled. The cup still has to look like a cup. The reference for a fold is the folded object. Hinge data can be consistent with itself and still finish as a wedge, a triangle, or a frog hidden by strips that only pretend to be cuts.

Agents report to one supervisor, and a person decides at the gates. I created the repository, because the agent could not. I dropped the first idea after it had already merged. I stopped the first brainstorm, and I stopped the second. I chose origami, and I chose to replace this repo. The merge is still mine, and this one is not accepted.

None of these rounds came with a traffic figure or a usage figure. The choice of origami was a judgment at a gate. It was not a measurement.

## References

- Repository [prompt-marketplace](https://github.com/rajeshradhakrishnanmvk/prompt-marketplace), created 15 October 2026. GitHub Pages serves the merged marketplace from `main`.
- [Pull request #1](https://github.com/rajeshradhakrishnanmvk/prompt-marketplace/pull/1), "Add a Jekyll prompt marketplace for GitHub Pages," merged 15 October 2026.
- [Pull request #2](https://github.com/rajeshradhakrishnanmvk/prompt-marketplace/pull/2), "Add the Foldly origami paint-and-fold site," draft, open, and not merged.
