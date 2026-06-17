# AI301-Github-Contribution-Log
This is a contribution README that tracks my entire journey from issue selection through pull request submission.

# Contribution [2]: [[FEATURE] Add a way to go to specific page of book #814]

**Contribution Number:** [1]  
**Student:** [Linda Mukundwa]  
**Issue:** [GitHub issue link](https://github.com/stumpapp/stump/issues/814)  
**Status:** [Phase II] [Complete]

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

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

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
