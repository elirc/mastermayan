# Code Review Mindset

## Review layers

1. Does it work?
2. Is it correct under real permissions and async timing?
3. Will it stay correct after the next maintainer touches it?
4. Does it fit the codebase’s existing boundaries?
5. Is it kind to future operators and reviewers?

## Repo-specific review checklist

- Does the change preserve ACL-filtered lookup patterns?
- Did it add hidden work inside model `save()` or signals?
- Did it change task signatures or callback kwargs without tests?
- Did it consider Docker/CI/runtime implications?

## Good review comments

> I think this changes the authorization boundary from filtered lookup to post-fetch check. Can we keep the `restrict_queryset` pattern so the object is never loaded outside the allowed set?

> This is readable, but it moves OCR-related work into the request path. Was that intentional, and do we have latency numbers showing it is safe?
