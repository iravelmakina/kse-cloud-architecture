# ADR-001: Access control model for multi-tenant buckets

**Status:** Accepted · **Date:** 2026-09-20 · **Author:** Iryna Velmakina

## Context

The capstone is a serverless key-value store sold as a service. Multiple tenants store
data in S3 and none of them may reach another's keys. I don't know how many tenants
there will be, and onboarding a new one has to be a runtime operation — not an
infrastructure change.

I tested three ways of enforcing this in Practice 2, in a live account.

## Options

**A. One bucket policy per tenant (RBAC).** The tenant's role is named as `Principal`
in the bucket policy. This works and it's easy to audit: open the policy, read who has
access. But every new tenant means a new role and a new policy. At 500 tenants that's
500 documents, and onboarding requires a deploy. Dead on arrival for a SaaS.

**B. Conditions on the request or the object.** One bucket, tenants separated by key
prefix (`s3:prefix`) or object tag (`s3:ExistingObjectTag`). Cheap, no per-tenant
resources. Two problems. A prefix is not a directory — S3's keyspace is flat, and the
condition filters a *request parameter*, not the data. When I ran `ls private/` the
denial refused the question; the object itself was never protected. And object tags can
be rewritten by anyone holding `s3:PutObjectTagging`, which makes the tag a weak place to
put a security decision.

**C. ABAC on a principal tag.** One policy for everyone, with
`${aws:PrincipalTag/tenant}` substituted into the resource ARN at evaluation time. I
verified this: `lab-atlas` and `lab-nebula` carry byte-identical policy documents and get
opposite results, decided purely by their tag. Adding a tenant changes zero policies.
The cost is that the decision moves out of the policy and into tag data — whoever can
set tags can grant themselves anything.

## Decision

**C — ABAC on a session tag.**

- Tenant identity comes from a session tag. The control plane calls `AssumeRole` with
  `--tags tenant=<id>` when issuing credentials.
- One shared policy, scoped by `arn:aws:s3:::<bucket>/tenants/${aws:PrincipalTag/tenant}/*`.
- Keys are namespaced `tenants/<tenant_id>/...` **because the tag has to resolve into a
  path.** The prefix is the mechanism ABAC runs on, not a second lock — both the prefix
  and the ARN check the same tag value, so if the tag is wrong they are wrong together.
  What it does buy is operational: a bug in the application that writes to the wrong path
  gets denied, ownership is readable from the key, and lifecycle rules and metrics can be
  scoped per tenant.
- An explicit `Deny` on `iam:TagRole`, `iam:UntagRole` and `sts:TagSession` for every
  principal except the control plane role. **This is the only control that fails
  independently of the tag, so it is the real second layer.**

The deciding factor is onboarding. Tenant count is unknown and a new customer cannot
require a deploy. That rules out A regardless of its other merits. B on its own was never
a candidate — a prefix condition filters a request parameter, so it can't be the isolation
boundary.

## Consequences

**Better.** One policy document instead of N. Onboarding is a tag, not a deploy. Adding
tenant 501 costs nothing. This matches the AWS pooled-isolation pattern, so I'm not
inventing anything unusual.

**Worse, and I'm accepting it deliberately:**

- **Single point of compromise, and only one real layer.** Option A gives 500
  independent locks; this gives one master key. Whoever controls tag issuance controls
  every tenant. The key prefix does not help here — it resolves from the same tag. The
  `Deny` on tagging APIs is the only control that fails separately, which makes it the
  most security-critical statement in the account. If it is ever removed or scoped wrong,
  isolation is gone with no second line behind it.
- **Auditing is no longer a document read.** "Who can read tenant X?" can't be answered
  from the policy. It needs the live tag state of every principal. I'll need CloudTrail
  on `AssumeRole` calls to reconstruct who held which tag when.
- **Naming becomes load-bearing.** The resource ARN is built from a tag value, so the key
  convention is a security control, not a style choice. The lab's own policy shows how
  this bites: `arn:aws:s3:::kse-*-${project}` lets any atlas-tagged role read *every*
  student's atlas bucket. My pattern has no wildcard before the tenant segment for
  exactly that reason.
- **No typo safety.** A misspelled tag fails silently — or worse, silently matches
  another tenant. AWS never shows the resolved value in the error: both denials I got in
  Task 3 said "no identity-based policy allows the action" even though the policy exists.
  Tag values need validation at issuance, because debugging them after the fact is blind.

## Verification

Prove it in Milestone 1: two tenant tags, one bucket, cross-read denied, and a check
that the tenant role cannot call `iam:TagRole` on itself.
