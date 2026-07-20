# AI301-Github-Contribution-Log
This is a contribution README that tracks my entire journey from issue selection through pull request submission.

# Contribution [2]: [[FEATURE] Add a way to go to specific page of book #814]

**Contribution Number:** [4]  
**Student:** [Linda Mukundwa]  
**Issue:** [GitHub issue link](https://github.com/stumpapp/stump/issues/814)  
**Status:** [Phase IV] [Pending Approval]

---

## Why I Chose This Issue

I chose this particular issue because I have never contibuted to an open-source project before and this seems like a fun challenge that can grow my skills as an engineer at the same time. I like the overall project as well as the tech stack. I have worked with the languages and frameworks mentioned before but I could brush up on them. Stump is a free and open source comics, manga, and digital book server with OPDS support, created with Rust, Axum, SeaORM and React. This blends with many interests of mine that I already have so it seems like a perfect project to start with. Additionally, there is very thorough documentation and multiple contributors, which make it even more fun. 

I am excited to challenge myself in a way that I have not done before. I am also excited to work on a project which involves mobile development because I find that super intriguing. Overall, I believe that this project will be a great way to think and behave like an engineer that's part of a team. It's a great transition from being a student to working in the industry and I cannot wait to see how much I grow!


---

## Understanding the Issue

### Problem Description

The main problem is more a feature that wants to be added to the web server. A nice feature to have is a way to scroll to a page that you were at previously to eliminate the need to continously scroll back, especially when you are reading a longer book or piece of media. Additionally, from the testing I have done, it seems to work differently depending on whether it is a web server, or mobile/desktop app. This makes coming up with the problem a bit difficult since the structure of the code works differently depending on each case. 

### Expected Behavior

The expectation is more of a nice ask. The GitHub issue is a request for a feature with an option or button to choose a specific page would significantly reduce recovery problems. If the application loses the book position then you are able to quickly jump back to the page you were reading.

### Current Behavior

Currently, there is no support for this specific feature, hence opening the issue.

### Affected Components

Out of scope: the EPUB reader (epub/) is location-based (epubcfi), not page-numbered — it already has LocationManager.tsx and a TOC for navigation, so a numeric "go to page" doesn't apply there. I'd keep this feature to the image-based reader (comics/manga/streamed PDF).


| Component | Role | Change |
| -------- | -------- | -------- |
| context.ts | Reader context | Read-only — already exposes setCurrentPage, book.pages, currentPage. No change needed. |
| ReaderFooter.tsx:183-190 | Renders the "X of Y" page text | Primary: make this indicator an interactive "go to page" control. |
| ReaderHeader.tsx:49-59 | 	Toolbar button row | Alt location: a "Go to page" control button could live here next to Settings. |
| GoToPage (new component) | The input/popover UI |New file under imageBased/container/. |
| paths.ts:54-83 | bookReader URL builder| No change — already supports ?page=N. |

---

## Reproduction Process

### Environment Setup

There was a large learning curve during this aspect for me because I had not previously worked with these libraries, and tools before so it was difficult to get the web app to work for myself. The docs were especially helpful because they provided helpful tools for Rust, Axum and OPDs. One more difficult aspect was getting Bacon to work with my VSCode. I kept having problems with the build running. 

I had not agreed to the Xcode license agreements and run sudo xcodebuild -license ... to review and agree to the Xcode and Apple SDKs license.

This is because rust uses Apple's cc/linker (from the Xcode Command Line Tools), and macOS blocks those tools until you've accepted the Xcode license. So every crate's build script fails to link with exit status: 69.

The fix was running this command in the terminal and accepting the license:

```bash
sudo xcodebuild -license accept
cargo run -p stump_server
```

### Steps to Reproduce

1. Read docs, clone repository and install dependencies
2. Run stump server and create library
3. Add media to read in library and preview on app
4. Scroll while reading, there won't be option to go back to certain page

### Reproduction Evidence

- **Commit showing reproduction:** N/A since this is a feature to implement.
- **Screenshots/logs:**
<img width="1459" height="786" alt="Screenshot 2026-06-16 at 23 30 30" src="https://github.com/user-attachments/assets/9ba0c467-31b1-423e-98d8-300281313478" />
<img width="1466" height="809" alt="Screenshot 2026-06-16 at 23 31 24" src="https://github.com/user-attachments/assets/56701f4a-b580-4411-8c67-493f8bcb8752" />

- **My findings:** 
The mechanism slightly varies depending on the type of app you are using, so finding a cross-platform approach is critical. Additionally, some browsers restricting certain viewing permission, which is another hurdle to cross.

---

## Solution Approach

### Analysis

As stated above, this is a feature, not a bug but the motivation is real. Reading position is already persisted server-side. BookReaderScene.tsx:18-25 fetches readProgress.page, and ImageBasedReader.tsx:46 seeds currentPage from initialPage. Progress is written back on every page change via ImageBasedReader.tsx:118-121.
So then, if that restore fails (incognito mode, a stale/missing readProgress, a URL opened without ?page=, or a sync hiccup), the user has no manual way to jump to an arbitrary page. They can only step forward/back or scrubbing the footer thumbnail strip. A "Go to page N" control is the manual-recovery escape hatch.
Thankfully, a lot of the hard part is already built because ImageBasedReader.tsx:109-124 defines handleChangePage, exposed through context as setCurrentPage. Calling it already (a) updates state, (b) rewrites the URL to ?page=N, and (c) reports progress. So a jump feature is almost entirely a UI addition that calls one existing function.

### Proposed Solution

Make the footer's existing "X of Y" indicator a clickable trigger that opens a small popover with a numeric input ("Go to page __ of N") and a submit. On submit, clamp the value to 1..book.pages and call setCurrentPage(target) by reusing the existing routing/progress plumbing. The footer is the right home because it already carries page context and the thumbnail scrubber, so the affordance is discoverable where users already look for position.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Add a control to the image-based reader letting the user jump directly to any page number, so a reader who has lost their place (or whose auto-restore failed) can immediately return to a known page.

**Match:** 
- setCurrentPage from context already does state + URL (?page=N) + progress write, the entire navigation mechanism. (ImageBasedReader.tsx:109-124)
- Clamping precedent exists: BookReaderScene.tsx:153-155 already redirects page > book.pages down to book.pages.
- @stump/components provides the Input/Popover/Text primitives already used across the toolbar (e.g. SettingsDialog, TimerMenu in the header).
- The footer already computes currentSet/book.pages for the "X of Y" label.

**Plan:** [Step-by-step implementation plan]
1. Create imageBased/container/GoToPage.tsx: a Popover triggered by a button; contains a numeric Input (default = currentPage, min=1, max=book.pages) and a submit handler. Pull currentPage, setCurrentPage, book from useImageBaseReaderContext().
2. In the handler, parse + clamp: Math.min(Math.max(1, n), book.pages); ignore NaN/empty; call setCurrentPage(clamped) and close the popover. Support Enter-to-submit.
3. Wire it into ReaderFooter.tsx:183-190 : wrap the existing "X of Y" Text as the popover trigger (keeps current visual, adds a hover/cursor affordance).
4. Add i18n keys for label/placeholder (the codebase uses useLocaleContext/t(...) — see locale usage in ServerConnectionErrorScene.tsx:28).
5. Tests: add GoToPage.test.tsx next to the existing container/__tests__/ suite, covering clamp-high, clamp-low, non-numeric rejection, and that submit calls setCurrentPage with the right value.

**Implement:** 
-  suggest git checkout -b feat/reader-go-to-page off main.
-  My current feature branch: [Feature Branch](https://github.com/LindaMukundwa/stump/tree/feat/reader-go-to-page)

**Review:**
1. Reuses setCurrentPage rather than duplicating URL/progress logic 
2. Respects readingMode (works in Paged; in continuous mode setCurrentPage still updates state, verify scroll-to behavior, see note below)
3. i18n strings, no hardcoded copy
4. Matches surrounding toolbar component style (forwardRef, motion, Tailwind classes)
5. Lint/types: yarn lint && yarn check-types


**Evaluate:** 

- Once server build is working, open a comic in Paged mode → open the footer → enter a page → confirm it navigates, the URL shows ?page=N, and reloading restores that page.
- Edge cases: enter 0, 99999, blank, letters → confirm clamping/no-op.
- Continuous mode caveat to test: in CONTINUOUS_* modes, handleChangePage updates currentPage but does not push to the URL (ImageBasedReader.tsx:111-116), and the continuous reader scrolls via initialPage. Verify whether setting currentPage actually scrolls the viewport there, or whether you need to thread a scroll-to into ContinuousScrollReader. This is the one area where "just call setCurrentPage" may not be sufficient.

Note: Still questioning whether the jump control should appear in both Paged and Continuous modes (requires the continuous-scroll wiring in step above), or ship Paged-only first and treat continuous as a follow-up? Leaning towards Paged-first to keep the initial PR small and avoid the scroll-sync complexity but worth mentioning in Stand up meeting.
---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [1] Progress

This week, I worked on creating the component feature itself by creating the file, GoToPage.tsx, which is a pure, prop-driven popover control (currentPage, totalPages, onSubmit). Validation is extracted into an exported clampPage() so the rule is testable without rendering. Debug logs are namespaced. I tested this file with unit tests found under GoToPage.test.tsx that has 10 tests covering clamp-high/low, empty book, non-numeric rejection, click-submit, and Enter-to-submit.

**Some of the test results are:**

yarn jest GoToPage → 10/10 pass
yarn check-types (browser) → clean, exit 0
What the tests already caught and fixed (the value of testing first):

An invalid variant="primary" on Button (only default/ghost/outline/secondary/destructive/link exist).
Two query ambiguities. The submit button "Go" and trigger "Go to page" both matched /go/i; resolved by matching the input via its spinbutton role and the submit button by exact name.
Adopted the linter's canonical text-gray-450 over the raw hex.

Since there are little errors, I am now working on wiring it into the footer itself. This swaps the static "X of 42" Text at ReaderFooter.tsx:183-190 for <GoToPage>, binding onSubmit to setCurrentPage from context and gating it to ReadingMode.Paged. That's where the [GoToPage] debug logs become observable and you can verify the URL updates to ?page=N in the browser.

Some of the challenges during this level where adhering to the strict code guidelines, using the correct libraries and determining how each component was connected in the application. As of now, there is low risk to the app running since nothing is directly calling on GoToPage.tsx but I am making sure to thoroughly test at each step.

I am also making 2 deliberate design choices that make this reviewer-friendly:

GoToPage is pure and prop-driven (currentPage, totalPages, onSubmit). This is exactly like the existing ImageScalingSelect.tsx (value/onChange). Context gets wired at the call site in Increment 2. This means the validation logic is unit-testable with no context/router mocking, and the component can't accidentally couple to reader internals.

Reuse, don't reinvent. onSubmit will be bound to the existing setCurrentPage (= handleChangePage), which already does state + URL ?page=N + progress write (ImageBasedReader.tsx:109-124). No new navigation code so a reviewer can see the jump goes through the same audited path as every other page change.

### Week [4-6] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:**
- * packages/browser/.../container/GoToPage.tsx 
  * packages/browser/.../container/__tests__/GoToPage.test.tsx
  * packages/browser/.../container/ReaderFooter.tsx
  * 
- **Key commits:** [Important Commits](https://github.com/LindaMukundwa/stump/commit/c3468bdd7cd26059c950ffe01a8c6a153b7052e0)
- **Approach decisions:**
- Not overengineering, adequately testing each step and functionality were at the forefront of each decision.

---

## Pull Request

**PR Link:** [Stump Pull Request](https://github.com/stumpapp/stump/pull/1294)

**PR Description:**

Closes [ISSUE #814 ](https://github.com/stumpapp/stump/issues/814)

## Description

Issue #814 -  feat(reader): go-to-page control, and fix PDFium binding reuse & thread-safety

## Summary

This PR contains two independent changes that came out of getting the reader working end-to-end during local setup:
1. Feature: a "go to page" control in the image-based reader for quick position recovery.
2. Demo: https://imgur.com/a/fg4fzLz 
3. Fix: corrects a PDFium binding bug (every PDF page after the first failed) and a latent thread-safety issue in the PDF renderer.

They're bundled because the PDF fix was a prerequisite to exercising the reader locally. Happy to split into two PRs if preferred and any other changes are requested. I am in a class learning to contribute to open-source projects, while being AI-native and critical of the output; therefore, AI assistance from Claude was used in part for this contribution. This is my first contribution to any open-source project ever, so I appreciate any advice and constructive criticism. Thank you!

## 1. Feature: "Go to page" control

**What & why??**

This was already an open issue. In paged mode, the footer's position indicator (e.g., "4-5 of 42") is now a clickable trigger that opens a small popover with a number input, letting the reader jump directly to any page. This is a manual-recovery affordance: if the reading position is ever lost, the user can type the page they remember and return to it immediately.
Scope
* Image-based reader, paged mode only. Continuous-scroll modes keep the plain-text indicator, since their page-change path doesn't sync to the URL the same way; this was deferred intentionally.
* EPUB reader unaffected (it's location/CFI-based with its own TOC + bookmarks).

**How it works**

* The new GoToPage component is presentation-only: it takes currentPage, totalPages, and an onSubmit callback. It owns no reader state and performs no navigation itself.
* In ReaderFooter, onSubmit is bound to the existing setCurrentPage handler, so a jump goes through the same state + URL (?page=N) + progress-write path as every other page change. No new navigation logic.
* Input is validated and clamped to 1..totalPages; non-numeric input is ignored. Submits on click or Enter.

**Files**

* packages/browser/.../container/GoToPage.tsx : new component + exported clampPage helper.
* packages/browser/.../container/__tests__/GoToPage.test.tsx: unit tests.
* packages/browser/.../container/ReaderFooter.tsx : wires the control in (paged mode), reusing the existing page-range label.

**Testing**

* 0 unit tests: clamp above/below range, empty book, non-numeric rejection, click-submit, Enter-submit, trigger rendering.
* yarn check-types + eslint clean on all changed files.
* Manually verified in browser: paged mode → click indicator → enter page → reader jumps, URL updates to ?page=N, progress persists.

## 2. Fix: PDFium binding reuse + thread-safety

The Problem (three issues, surfaced in order)

1. PdfConfigurationError : pdfium_path was unset and no system PDFium existed. (Resolved locally by installing libpdfium.dylib and pointing config at it, environment/config, no code change.)
2. PdfiumLibraryBindingsAlreadyInitialized : a real code bug. pdfium-render stores its loaded library in a process-global cell that can only be initialized once, but renderer() called bind_to_library() on every render, so the second page ever viewed failed. pdf_prerender_range = 5 made it fail instantly by rendering several pages concurrently.
3. Latent thread-safety: PDFium's native library isn't thread-safe, but renders run across multiple spawn_blocking threads.

Fix: core/src/filesystem/media/format/[pdf.rs](http://pdf.rs/)

* Added a process-global PDFIUM_LOCK: Mutex<()> that serializes all PDFium access.
* Rewrote renderer() to bind once and reuse existing bindings afterward - on PdfiumLibraryBindingsAlreadyInitialized it falls back to Pdfium::default(), which the crate documents as reusing already-initialized bindings. It now returns a MutexGuard held for the whole render.
* Updated the three call sites (get_page_count, page render, and the file converter) to keep the guard alive via let (_guard, pdfium) = ….

**Testing**

* Verified locally: PDFs now render past the first page, including with pdf_prerender_range = 5 (previously an instant failure); no binding-reinit errors under concurrent prerendering.

Setup note for reviewers/testers
To exercise PDF rendering you need a PDFium binary and pdfium_path set to it (macOS: libpdfium.dylib). (Possible follow-up: document this in docs/.../installation/source.mdx.)

## Screenshots

https://imgur.com/a/fg4fzLz

## Ready?

Please read each item and check the boxes:

- [☑] I read the [contributing guidelines](https://github.com/stumpapp/stump/blob/main/.github/CONTRIBUTING.md)
- [☑] I searched for existing issues or pull requests that may be related to my contribution
- [☑] This PR is based into `nightly` and not `main`
- [☑] I added tests and/or documentation for my changes if applicable

## Stump Contributor License Agreement

By contributing to Stump, you agree that your contributions will be licensed under the following licenses (where applicable):

- The [expo application](https://github.com/stumpapp/stump/blob/main/apps/expo/LICENSE) is licensed under [GPL-3.0](https://www.gnu.org/licenses/gpl-3.0.html)
- All other code in the repository is licensed under [MIT License](https://www.tldrlegal.com/license/mit-license)

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
