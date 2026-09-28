# RFCs

Every format here is read and written by tools that do not live in this repository, so a
change to one commits all of them. A pull request that changes what a reader or writer of a
format has to do gets the `rfc` label and cannot merge for seven days, so the people
implementing it can comment first. That includes a new packet type: old readers skip it, but
new writers will emit it and new readers will be expected to understand it.

Fixing wording, typos or an example, without changing what a tool does, does not need one.
Who opens the pull request makes no difference.

## The window

While the label is on, the `rfc-moratorium` check fails until seven days after the label was
last applied, and `main` requires that check. The check's description gives the time the
window closes. If the proposal changes during the window, remove and re-apply the label and say
what changed in a comment; the seven days start over.

Open RFCs are the
[open pull requests with the label](https://github.com/ps1dev/standards/pulls?q=is%3Apr+is%3Aopen+label%3Arfc).
When the label goes on, the people listed in [.github/rfc-stakeholders](.github/rfc-stakeholders)
are mentioned on the pull request.

## What an RFC has to say

Before the window closes, the pull request says which tools will read and write the change,
and links an implementation of at least one side. An RFC missing that does not merge when the
window closes.

## Commenting

Comment on the pull request. An objection helps most with a use case attached: what your tool
does today that the change would break, or what it would stop you from doing.
