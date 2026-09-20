# Practice 1

**Student:** Iryna Velmakina · **User**: kse-student · **Account:** 361577816422 · **Region:** eu-central-1 · **Date:** 2026-09-20
**Bucket prefix:** `kse-velmakina-*`

Three access-control models tested against the same S3 data: RBAC (the role decides),
PBAC (the request context decides), ABAC (the principal's tag decides).

---

## 0. Environment

Five CLI profiles. Only `student` holds credentials; the other four assume roles via
`source_profile = student`, so every request carries an explicit identity.

```ini
# ~/.aws/config
[profile student]
region = eu-central-1

[profile hr]
role_arn       = arn:aws:iam::361577816422:role/lab-hr
source_profile = student
region = eu-central-1
# ... analyst / atlas / nebula identical, differing only in role_arn
```

### Identity check

```console
$ aws sts get-caller-identity --profile student
{
    "UserId": "AIDAVIL5XUFTPXKBWXIBD",
    "Account": "361577816422",
    "Arn": "arn:aws:iam::361577816422:user/kse-student"
}

$ aws sts get-caller-identity --profile hr
{
    "UserId": "AROAVIL5XUFTC6T5CFRKO:botocore-session-1789919747",
    "Account": "361577816422",
    "Arn": "arn:aws:sts::361577816422:assumed-role/lab-hr/botocore-session-1789919747"
}
```

`student` is a permanent IAM user. `hr` is an `assumed-role` session — a temporary
identity with its own session name, borrowed for the session and then gone.

---

## Task 1 — RBAC: the role decides

### Setup

```console
$ aws s3 mb s3://kse-velmakina-hr --profile student
make_bucket: kse-velmakina-hr

$ echo "salary data" > salaries.txt
$ aws s3 cp salaries.txt s3://kse-velmakina-hr/ --profile student
upload: ./salaries.txt to s3://kse-velmakina-hr/salaries.txt
```

### Policy — `policies/01-rbac.json`

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "OnlyHrRoleCanRead",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::361577816422:role/lab-hr"},
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::kse-velmakina-hr",
      "arn:aws:s3:::kse-velmakina-hr/*"
    ]
  }]
}
```

```console
$ aws s3api put-bucket-policy --bucket kse-velmakina-hr \
    --policy file://policies/01-rbac.json --profile student
$ aws s3api get-bucket-policy --bucket kse-velmakina-hr --profile student
{ "Policy": "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Sid\":\"OnlyHrRoleCanRead\", ... }]}" }
```

### Results

```console
$ aws s3 ls s3://kse-velmakina-hr --profile hr
2026-09-20 19:02:30         12 salaries.txt

$ aws s3 ls s3://kse-velmakina-hr --profile analyst
An error occurred (AccessDenied) when calling the ListObjectsV2 operation:
User: arn:aws:sts::361577816422:assumed-role/lab-analyst/botocore-session-1789921596
is not authorized to perform: s3:ListBucket on resource: "arn:aws:s3:::kse-velmakina-hr"
because no identity-based policy allows the s3:ListBucket action

$ aws s3 cp s3://kse-velmakina-hr/salaries.txt - --profile hr
salary data
```

| identity | operation | result |
|---|---|---|
| `lab-hr` | ListBucket | allowed |
| `lab-analyst` | ListBucket | AccessDenied |
| `lab-hr` | GetObject | allowed — `salary data` |

### Defense question — `lab-hr`'s own policy grants nothing on data. Where is the access from?

An S3 request is evaluated against **both** the identity-based policy attached to the
principal and the resource-based policy attached to the bucket. **Within a single
account the Allows union: an explicit Allow in either one is sufficient.**
`lab-hr`'s identity policy is silent, but the bucket policy names the role as
`Principal` and allows `s3:GetObject` and `s3:ListBucket`, so the request finds its
Allow there. `lab-analyst` appears in neither, so it stays at the default implicit deny
— which is exactly what the error states.

**Where this stops being true: cross-account.** If the principal lives in a different
account from the bucket, an Allow in the bucket policy alone is not enough. Both
accounts must allow the action independently — the resource owner's policy *and* the
principal's identity policy in their own account. Neither account can unilaterally grant
access across the boundary.

---

## Task 2a — PBAC: the request condition decides

### Setup

```console
$ aws s3 cp salaries.txt s3://kse-velmakina-hr/private/ --profile student
upload: ./salaries.txt to s3://kse-velmakina-hr/private/salaries.txt

$ echo "menu" > menu.txt
$ aws s3 cp menu.txt s3://kse-velmakina-hr/public/ --profile student
upload: ./menu.txt to s3://kse-velmakina-hr/public/menu.txt

$ aws s3 ls s3://kse-velmakina-hr --recursive --profile student
2026-09-20 19:32:18         12 private/salaries.txt
2026-09-20 19:33:52          5 public/menu.txt
2026-09-20 19:02:30         12 salaries.txt
```

### Policy — `policies/02a-prefix.json` (replaces Task 1's policy entirely)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListOnlyPublicPrefix",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::361577816422:role/lab-hr"},
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::kse-velmakina-hr",
      "Condition": {"StringLike": {"s3:prefix": "public/*"}}
    },
    {
      "Sid": "ReadOnlyPublicObjects",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::361577816422:role/lab-hr"},
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::kse-velmakina-hr/public/*"
    }
  ]
}
```

### Results

```console
$ aws s3 ls s3://kse-velmakina-hr/public/ --profile hr
2026-09-20 19:33:52          5 menu.txt

$ aws s3 ls s3://kse-velmakina-hr/private/ --profile hr
An error occurred (AccessDenied) when calling the ListObjectsV2 operation:
User: arn:aws:sts::361577816422:assumed-role/lab-hr/botocore-session-1789922668
is not authorized to perform: s3:ListBucket on resource: "arn:aws:s3:::kse-velmakina-hr"
because no identity-based policy allows the s3:ListBucket action

$ aws s3 cp s3://kse-velmakina-hr/private/salaries.txt - --profile hr
download failed: ... An error occurred (403) when calling the HeadObject operation: Forbidden

$ aws s3 cp s3://kse-velmakina-hr/public/menu.txt - --profile hr
menu
```

| command | result | deciding mechanism |
|---|---|---|
| `ls public/` | allowed | `ListOnlyPublicPrefix` — condition `s3:prefix` matched |
| `ls private/` | AccessDenied | same statement, **condition failed** → no statement applies |
| `cp private/salaries.txt` | 403 | `ReadOnlyPublicObjects` — **ARN did not match** (no condition involved) |
| `cp public/menu.txt` | allowed | `ReadOnlyPublicObjects` — ARN matched |

---

## Task 2b — PBAC: the object's tag decides

### Setup

```console
$ aws s3api put-object --bucket kse-velmakina-hr --key report-open.txt \
    --body menu.txt --tagging "classification=public" --profile student
{ "ETag": "\"423925c43ce3657f4bc68a50f808cddc\"", "ChecksumCRC64NVME": "CPG0gLwJ05M=",
  "ChecksumType": "FULL_OBJECT", "ServerSideEncryption": "AES256" }

$ aws s3api put-object --bucket kse-velmakina-hr --key report-closed.txt \
    --body salaries.txt --tagging "classification=secret" --profile student
{ "ETag": "\"d2c43f2c4f8cc1ad0b421c4455d4427e\"", ... "ServerSideEncryption": "AES256" }

$ aws s3api get-object-tagging --bucket kse-velmakina-hr --key report-open.txt --profile student
{ "TagSet": [ { "Key": "classification", "Value": "public" } ] }

$ aws s3api get-object-tagging --bucket kse-velmakina-hr --key report-closed.txt --profile student
{ "TagSet": [ { "Key": "classification", "Value": "secret" } ] }
```

### Third statement added to `policies/02b-tag.json`

```json
{
  "Sid": "AnalystReadsOnlyPublicClassified",
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::361577816422:role/lab-analyst"},
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::kse-velmakina-hr/*",
  "Condition": {
    "StringEquals": {"s3:ExistingObjectTag/classification": "public"}
  }
}
```

### Results

```console
$ aws s3 cp s3://kse-velmakina-hr/report-open.txt - --profile analyst
menu

$ aws s3 cp s3://kse-velmakina-hr/report-closed.txt - --profile analyst
download failed: s3://kse-velmakina-hr/report-closed.txt to -
An error occurred (403) when calling the HeadObject operation: Forbidden

$ aws s3 cp s3://kse-velmakina-hr/report-closed.txt - --profile hr
download failed: s3://kse-velmakina-hr/report-closed.txt to -
An error occurred (403) when calling the HeadObject operation: Forbidden

$ aws s3 ls s3://kse-velmakina-hr --profile analyst
An error occurred (AccessDenied) when calling the ListObjectsV2 operation:
User: arn:aws:sts::361577816422:assumed-role/lab-analyst/botocore-session-1789921596
is not authorized to perform: s3:ListBucket on resource: "arn:aws:s3:::kse-velmakina-hr"
because no identity-based policy allows the s3:ListBucket action
```

### Required: exact denial text and the responsible statement

| # | command | exact error | cause |
|---|---|---|---|
| 1 | `analyst` → `report-open.txt` | *(none — returned `menu`)* | `AnalystReadsOnlyPublicClassified` matched: ARN `/*` covers the key, tag = `public` |
| 2 | `analyst` → `report-closed.txt` | `An error occurred (403) when calling the HeadObject operation: Forbidden` | **Same statement, condition failed.** Tag is `secret` ≠ `public`, so the statement does not apply → implicit deny |
| 3 | `hr` → `report-closed.txt` | `An error occurred (403) when calling the HeadObject operation: Forbidden` | **Different cause.** `ReadOnlyPublicObjects` scopes to `public/*` by ARN; the key is at root. No condition involved — no statement matched |
| 4 | `analyst` → `ls` bucket | `An error occurred (AccessDenied) when calling the ListObjectsV2 operation: User: arn:aws:sts::361577816422:assumed-role/lab-analyst/… is not authorized to perform: s3:ListBucket on resource: "arn:aws:s3:::kse-velmakina-hr" because no identity-based policy allows the s3:ListBucket action` | **Absence of a statement.** Nothing in the policy grants `lab-analyst` `s3:ListBucket` at all |

## Task 3 — ABAC: the principal's tag decides

No bucket policy. `lab-atlas` and `lab-nebula` carry byte-identical identity policies:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AbacOwnProjectBuckets",
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::kse-*-${aws:PrincipalTag/project}",
      "arn:aws:s3:::kse-*-${aws:PrincipalTag/project}/*"
    ]
  }]
}
```

### The roles differ only by tag

```console
$ aws iam list-role-tags --role-name lab-atlas --profile student
{ "Tags": [ { "Key": "project", "Value": "atlas" } ] }

$ aws iam list-role-tags --role-name lab-nebula --profile student
{ "Tags": [ { "Key": "project", "Value": "nebula" } ] }
```

### Setup

```console
$ aws s3 mb s3://kse-velmakina-atlas  --profile student
make_bucket: kse-velmakina-atlas
$ aws s3 mb s3://kse-velmakina-nebula --profile student
make_bucket: kse-velmakina-nebula
$ aws s3 cp menu.txt s3://kse-velmakina-atlas/  --profile student
upload: ./menu.txt to s3://kse-velmakina-atlas/menu.txt
$ aws s3 cp menu.txt s3://kse-velmakina-nebula/ --profile student
upload: ./menu.txt to s3://kse-velmakina-nebula/menu.txt
```

### Results

```console
$ aws s3 ls s3://kse-velmakina-atlas --profile atlas
2026-09-20 19:59:35          5 menu.txt

$ aws s3 ls s3://kse-velmakina-nebula --profile atlas
An error occurred (AccessDenied) when calling the ListObjectsV2 operation:
User: arn:aws:sts::361577816422:assumed-role/lab-atlas/botocore-session-1789923704
is not authorized to perform: s3:ListBucket on resource: "arn:aws:s3:::kse-velmakina-nebula"
because no identity-based policy allows the s3:ListBucket action

$ aws s3 ls s3://kse-velmakina-atlas --profile nebula
An error occurred (AccessDenied) when calling the ListObjectsV2 operation:
User: arn:aws:sts::361577816422:assumed-role/lab-nebula/botocore-session-1789925125
is not authorized to perform: s3:ListBucket on resource: "arn:aws:s3:::kse-velmakina-atlas"
because no identity-based policy allows the s3:ListBucket action

$ aws s3 ls s3://kse-velmakina-nebula --profile nebula
2026-09-20 19:59:44          5 menu.txt
```

### Matrix

| command | prediction | actual         |
|---|---|----------------|
| `aws s3 ls s3://kse-velmakina-atlas --profile atlas` | allowed — `${project}` → `atlas`, pattern `kse-*-atlas` matches | *allowed*      |
| `aws s3 ls s3://kse-velmakina-nebula --profile atlas` | AccessDenied — pattern `kse-*-atlas` does not match `…-nebula` | *AccessDenied* |
| `aws s3 ls s3://kse-velmakina-atlas --profile nebula` | AccessDenied — pattern `kse-*-nebula` does not match `…-atlas` | *AccessDenied* |
| `aws s3 ls s3://kse-velmakina-nebula --profile nebula` | allowed | *allowed*      |

### Finale — scaling

```console
$ aws s3 mb s3://kse-velmakina-orion --profile student
make_bucket: kse-velmakina-orion

$ aws s3 ls s3://kse-velmakina-orion --profile atlas
An error occurred (AccessDenied) when calling the ListObjectsV2 operation:
User: arn:aws:sts::361577816422:assumed-role/lab-atlas/botocore-session-1789923704
is not authorized to perform: s3:ListBucket on resource: "arn:aws:s3:::kse-velmakina-orion"
because no identity-based policy allows the s3:ListBucket action
```

### Closing question — 500 projects: how many policies change?

| | policies to change when adding project #501 | total policy documents to maintain |
|---|---|---|
| **RBAC** | 1 new role + 1 new policy, plus every existing policy that enumerates projects | ~500, growing linearly |
| **ABAC** | **0** | **1** |

ABAC is not free. The decision moved out of the policy document and into **tag data**:

- **Governance shifts to tagging permissions.** Whoever holds `iam:TagRole`
  (or `iam:TagUser`, or `sts:TagSession` / the IdP attribute mapping for federated
  principals) can grant themselves any project's data by editing one tag. One policy to
  review replaces 500 — but one very attractive target replaces 500 low-value ones.
  The same applies to `s3:PutObjectTagging` for the Task 2b model.
- **Auditing gets harder.** "Who can read project X?" can no longer be answered by
  reading a policy. It requires joining the policy against the live tag state of every
  principal.
- **Naming becomes a security control.** The `kse-*` wildcard here means `lab-atlas`
  reaches **every student's** `-atlas` bucket in this shared account, not only
  `kse-velmakina-atlas`. Under ABAC the resource naming convention is load-bearing;
  a typo or a permissive wildcard silently widens access.
- **No typo safety.** A misspelled tag yields silent denial (or, worse, a silent grant
  if it collides with another project's name).

---
## Files

```
lab1/
├── README.md
├── menu.txt
├── salaries.txt
└── policies/
    ├── 01-rbac.json       RBAC        — principal named in bucket policy
    ├── 02a-prefix.json    PBAC/prefix — condition on s3:prefix + ARN scoping
    └── 02b-tag.json       PBAC/tag    — condition on s3:ExistingObjectTag
```
