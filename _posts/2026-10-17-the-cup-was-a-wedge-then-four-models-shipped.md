---
layout: post
date: 2026-10-17 08:00:00 +0530
tags:
  - agents
  - architecture
  - github-pages
  - lessons
---

# The Cup Was a Wedge, Then Four Models Shipped: Lessons from Foldly

On 15 October 2026 I asked for a Jekyll site that GitHub Pages could build from vanilla HTML, CSS, and JavaScript, with no third-party dependencies. Pages has no server and no database, so the site had to live in the repository and run in the browser. I created [prompt-marketplace](https://github.com/rajeshradhakrishnanmvk/prompt-marketplace) myself. The GitHub connection on the agent could not create repositories, and Cursor's access to the new repo had to be granted after the repo existed.

The product I asked for was a prompt marketplace with a feed in the style of LinkedIn or Facebook. A visitor could add a prompt, put it in a category, and filter the list. There is nowhere to post a form, so each prompt is a Markdown file in `_prompts`, and the way onto the site is a pull request. The Add Prompt form opens GitHub's new-file page with the filename and the body already filled in.

## The feed the notes and GitHub disagree about

A Cursor cloud agent built that in [pull request #1](https://github.com/rajeshradhakrishnanmvk/prompt-marketplace/pull/1). Ten seed prompts. A Liquid include that writes a JSON index. Filters and search kept in the URL query string, so a filtered view has a permalink. `jekyll build --safe` passed.

The architect's notes said this pull request was never merged. GitHub shows a merge. The state I can verify is `merged` at 15:03 UTC on 15 October 2026, merge commit `f40b480`, merged by my account, into `main`. Pages built that commit. On the morning of 16 October I still did not want the product. I had no answer for why a stranger would open the tab.

I asked how a site inside these constraints gets traffic. The architect supervising the work proposed four directions: a before-and-after gallery, a daily challenge, a niche library, and a template playground. I then had that architect create Crew73, six agents under its supervision: Trend Scout, Growth Strategist, Product Designer, Builder, Content Creator, and QA Critic.

## Three rounds, then a square of paper

The first brainstorm used material we already had. A daily data-engineering puzzle, Schemadle. A Stephen King puzzle. A Lakehouse playground. A quiz. I said to drop data engineering and Stephen King and go niche. The second round came back India-focused: an RTO-code map game, a lost-unit converter, and ham radio exam practice. I asked them to think more. The third round produced global micro-niches and what the brief called "wow" toys. QA Critic's top pick was an origami paint-and-fold toy. I picked it, and I chose to keep this repository and replace its contents.

Product Designer wrote the spec and named it Foldly. The sheet is a CSS 3D hinge tree. The flaps are `clip-path` shapes. There is no fold solver. The folds are keyframes. Painting is Canvas 2D, and a stroke can sit on the front or the back of the square. A share URL runs the strokes through `CompressionStream` deflate and encodes them as base64url. PNG and WebM export re-renders that picture to a canvas and records it with `MediaRecorder`. Each model has one page for search and for print. Content Creator wrote the copy, renamed the fortune hat to a samurai helmet (kabuto, id `samurai-helmet`), and labelled estimates and unknowns as estimates and TBDs.

## What the first draft failed

Builder opened [pull request #2](https://github.com/rajeshradhakrishnanmvk/prompt-marketplace/pull/2) as a draft. The first commit, `6276841`, put six models in the tree: a paper cup, a cat face, a dog face, a tulip with a flat head and no stem, the samurai helmet, and a jumping frog lite. QA Critic tested that draft with Playwright on Chromium, WebKit, an iPhone 13, and a Pixel 7. The verdict was do not ship.

In WebKit the painted front faces never rendered. `clip-path` inside `preserve-3d`, together with `backface-visibility`, dropped the paint. The builder had noted that Safari was not run on the Linux VM that produced the draft. A `display` rule overrode the `[hidden]` attribute. The layout overflowed on the phones. Placeholder TBD copy was still on the page. The first folds moved only the front layer, so none of the six finished models was recognisable. The cup was a flat wedge. The cat and the helmet were triangles. The frog was hidden by fake "cut" strips. Smaller gaps sat beside those: no `hashchange` listener, short contrast, no `og:image`, and the print scale.

<img src="/assets/images/foldly-pivot-agent-flow.svg" alt="Flow from the merged prompt marketplace through a refused six-model Foldly draft to five commits and a four-model site merged on 17 October 2026" title="Pull request #1 is merged. Pull request #2 merged on 17 October with the cup, cat, dog, and tulip. The frog and the samurai helmet are not on the site." style="width:100%; height:auto;">

## What changed to ship

I merged pull request #2 at 02:01 UTC on 17 October 2026. GitHub records the merge on my account. The commit on `main` is `e570ce5`, a squash whose parent is `6f0fe0c`. Pages built that commit, and the project site is up.

I opened [https://rajeshradhakrishnanmvk.github.io/prompt-marketplace/](https://rajeshradhakrishnanmvk.github.io/prompt-marketplace/) after that build. The title is "Foldly — Paint a square. Watch it fold." The home page lists four models: paper cup, cat face, dog face, and tulip. Those four model URLs return 200. `/models/jumping-frog-lite/` and `/models/samurai-helmet/` return 404. `main` has JSON and Markdown for the same four, and none for the frog or the helmet. The paper-cup page labels difficulty and time as estimates.

The description on the pull request still says the pull request is not merged. I am using the merge record and the Pages build, which agree with each other.

Five commits separate the refused draft from that squash. The note on the last of them titles the pass "QA round 4". The commit count and that label are both in the pull request. They do not use the same index.

| Commit | What landed |
| --- | --- |
| `6276841` | The six-model draft QA refused. |
| `981dee5` | The frog and the helmet come out. The commit message says the waterbomb frog and the kabuto horns cannot be made with one hinge per flap, and that the cup, cat, dog, and tulip now finish on the shapes their steps describe. |
| `e46b290` | The painted fold is drawn on a canvas, and the placeholder sentences come out. The merged `fold-view.js` says WebKit drops images and gradients on CSS 3D planes, so both browsers use that canvas. |
| `ca0e253` | Each fold is fitted inside the frame. `frameFit` takes a margin of `0.06`. The same commit aims share links at about 1,800 characters. |
| `78fb686` | A share link is refused when a faithful copy still will not fit in 1,800 characters. `encodePayload` returns no token once both the stroke encoding and the JPEG fallback are over that length. The share sheet then says to download a picture or a video. The pull request calls this pass QA round 4. |

The scope that shipped is that cut. The live site has the four models the second commit left in the tree. I have no traffic figure and no usage figure for any of this.

## What I am keeping

Validate the idea before the build. Pull request #1 met the brief I wrote, and I merged it. Pages served it. I still had no reason for a visit on the morning of 16 October. The cheaper moment to say so was before the agent opened the pull request.

The constraint is part of the design. With no backend, the site has no accounts, no live write, and no feed except a rebuild. Foldly puts the interaction in the browser: paint on a canvas, a compressed URL as the save file, and a Jekyll page per model. The same limit that made the marketplace awkward is the limit this toy inhabits. The toy that is actually on Pages is four models, after two were removed.

An adversarial reader with a real browser sees what the author's own checks skip. The first verdict was do not ship, and the failures were the ones a visitor would have seen: a missing texture in WebKit, a cup that finished as a wedge, a fold that left the frame, a share link with no size limit. The later commits are the record of answering those failures. A passing `jekyll build` on the first draft did not make the cup a cup.

Agents report to one supervisor, and a person decides at the gates. I created the repository, because the agent could not. I dropped the marketplace as an idea after I had already merged it. I stopped the first brainstorm, and I stopped the second. I chose origami. I merged Foldly on 17 October, after the frog and the helmet were out of the tree. That merge is the gate.

## References

- Repository [prompt-marketplace](https://github.com/rajeshradhakrishnanmvk/prompt-marketplace), created 15 October 2026.
- [Pull request #1](https://github.com/rajeshradhakrishnanmvk/prompt-marketplace/pull/1), "Add a Jekyll prompt marketplace for GitHub Pages." GitHub shows it merged at 15:03 UTC on 15 October 2026, commit `f40b480`. The architect's notes said it was never merged.
- [Pull request #2](https://github.com/rajeshradhakrishnanmvk/prompt-marketplace/pull/2), "Add the Foldly origami paint-and-fold site." GitHub shows it merged at 02:01 UTC on 17 October 2026, squash commit `e570ce5`.
- The project site Pages built from that commit: [https://rajeshradhakrishnanmvk.github.io/prompt-marketplace/](https://rajeshradhakrishnanmvk.github.io/prompt-marketplace/).
