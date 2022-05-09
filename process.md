# **Process for introducing new guide documents**

1. First, a draft is developed outside this repository among authors. To avoid
   duplication of effort, authors may open an issue. The title should start
   with "In progress:".

1. Towards the end of the draft phase, at the point the authors wish to engage
   wider collaboration, the authors may open a PR with the draft document
   against the `draft/` directory. This PR will be merged by repository
   maintainers without review. At this time the above "In progress:" issue
   should be closed.

1. The document may live in `draft/` for as long as the authors
   wish. Repository maintainers will merge any PRs from authors on these drafts
   without review.

1. When the authors are ready to submit the draft for formal review, they open
   a PR to move the draft to the `review/` directory. At this point an RFC
   announcement is made to the Working Group. This starts a 30-day count
   down. An issue is created (Title: "RFC for Foo guide closes yyyy-mm-dd")
   under a new Milestone with the due-date 30 days away.

1. During the review time, comments are taken in the form of issues. The title
   should start with "RC Foo:", where Foo is the name of the guide.

1. Authors address review comments by making PRs to the guide and closing out
   the comments.

1. After the 30 day period is over, the guide will be moved from the `review/`
   to the `published/` directory once all issues are resolved. The guide will
   be given the version 1.0

## **Process for maintaining published guides**

* Repository maintainers will regularly triage incoming issues on published
  guides with the tags:

  * Minor: The fix is obvious or clear. Does not need in-depth review and RFC
    period. (e.g., typo)

  * Major: Improvements to guide that require more in-depth expertise.

  * Critical: Problems that cause security issues when following guide as is.

* At least every 3 months, repository maintainers will address all minor issues
  on guides, then bump the minor version.

* Major issues will be postponed until available authors can take on a major
  version update. This will follow the same workflow as new guides above.

* Critical issues will be floated up to WG discussion, and a call for help made
  there.
