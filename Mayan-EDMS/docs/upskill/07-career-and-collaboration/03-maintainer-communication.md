# Maintainer Communication

## Ask for help without outsourcing thinking

- include exact reproduction
- include files and lines already inspected
- include your current hypothesis and what disproved alternatives you checked

## Templates

### Asking a question

I traced the issue through `path:line-line` and `path:line-line`. My current hypothesis is `X` because `Y`. Before I go further, can you confirm whether `Z` is an intentional invariant?

### Proposing a feature

I think this improves `flow` with low blast radius because it follows the existing pattern in `path:line-line`. Risks I see are `A` and `B`. Proposed test coverage: `...`

### Reporting a bug

Environment:
Steps:
Expected:
Actual:
Suspected boundary:
Related anchors:

### Responding to requested changes

Thanks. I agree on `A`; I missed the existing pattern in `path:line-line`. I’ve updated the implementation and added coverage for `B`. I left `C` unchanged because `reason`.
