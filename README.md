# S3 Security Lab: Testing the CIA Triad

Most cloud breaches don't come from sophisticated zero-day exploits, they come from simple misconfigurations. To understand that firsthand, I built a small hands-on lab in AWS to test the three pillars of cloud security: **Confidentiality, Integrity, and Availability** (the CIA triad).

This README documents what I built, the commands I ran, the mistakes I made along the way, and what I learned from each one.

## Setup

Before writing any commands, I verified my AWS CLI configuration:

```
aws sts get-caller-identity
```

This confirms who you're authenticated as, it returns your Account ID and User ARN, confirming your credentials are working.

**A gap I caught:** explaining my ARN to myself, I realized I was mixing up *resources* and *services*. An ARN is built from six colon-separated parts (partition, service, region, account ID, resource type/name), and I'd been treating the service name (like `s3` or `ec2`) and the resource name (like `user/fatemeh`) as the same thing. They're not, the service is *what AWS product* the resource belongs to; the resource is the *specific thing itself*.

I created the S3 bucket:

```
aws s3api create-bucket --bucket test2-security-s3 --region ca-central-1 --create-bucket-configuration LocationConstraint=ca-central-1
```

Note: `s3api` (not just `s3`) gives lower-level control needed for region configuration. The `--create-bucket-configuration LocationConstraint=...` flag is required for any region other than `us-east-1`, a quirk from how S3 was originally built.

## Confidentiality: Can a Stranger Read My Data?

Confidentiality is about keeping data private from anyone who isn't supposed to see it.

First, I checked whether Block Public Access was enabled on the bucket:

```
aws s3api get-public-access-block --bucket test2-security-s3
```

```json
{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": true,
        "IgnorePublicAcls": true,
        "BlockPublicPolicy": true,
        "RestrictPublicBuckets": true
    }
}
```

**A gap I caught:** I initially mixed up Network ACLs (a VPC/subnet-level networking concept) with S3 ACLs (an older, legacy bucket-level permission system), two unrelated things that share an acronym. Once I sorted that out, the four settings made sense as two pairs:

- `BlockPublicAcls` / `BlockPublicPolicy`, stop anyone from *creating* a new public ACL or bucket policy.
- `IgnorePublicAcls` / `RestrictPublicBuckets`, even if a public ACL or policy *already exists*, AWS ignores it and treats it as if it isn't there.

Block handles the future; Ignore/Restrict handles anything that already slipped through.

To test it, I uploaded a file and then tried to access it the way an external attacker would, no AWS credentials, just a plain HTTP request via `curl`:

```
curl https://test2-security-s3.s3.ca-central-1.amazonaws.com/confidential_test.txt
```

**Result:**
```
curl : The remote server returned an error: (403) Forbidden.
```

The file existed and the URL was correct, but without proper credentials, it was completely unreachable. Confidentiality: confirmed.

## Integrity: Can I Prove Data Wasn't Tampered With?

Integrity means making sure data hasn't been changed without you knowing. To test this, I simulated an attacker overwriting a file, then proved I could recover the original.

**Enable versioning:**
```
aws s3api put-bucket-versioning --bucket test2-security-s3 --versioning-configuration Status=Enabled
```

**Verify it's enabled:**
```
aws s3api get-bucket-versioning --bucket test2-security-s3
```
```json
{ "Status": "Enabled" }
```

**Simulate the attack**, upload an original file, then overwrite it with corrupted content:
```
echo "ORIGINAL SECRET DATA" > secret.txt
aws s3 cp secret.txt s3://test2-security-s3/secret.txt

echo "CORRUPTED DATA" > secret.txt
aws s3 cp secret.txt s3://test2-security-s3/secret.txt
```

Listing the object's version history confirmed both versions existed, the corrupted one marked `"IsLatest": true`, and the original still sitting underneath it, fully intact.

**Recovery, attempt one:** I tried downloading the original version by its Version ID.

**A mistake I made:** I initially used the *Owner ID* instead of the *Version ID*, both are long random strings, but they mean completely different things. Once I corrected this:

```
aws s3api get-object --bucket test2-security-s3 --key secret.txt --version-id <original-version-id> recovered.txt
```

The response confirmed the size and Version ID matched the original, and `"ServerSideEncryption": "AES256"` confirmed S3 encrypts data at rest. Opening the downloaded file confirmed it read exactly: `ORIGINAL SECRET DATA`.

**Recovery, attempt two, restoring in place:** instead of just downloading a copy, I copied the old version onto itself, creating a *new* current version with the original content:

```
aws s3api copy-object --bucket test2-security-s3 --key secret.txt --copy-source "test2-security-s3/secret.txt?versionId=<original-version-id>"
```

The result had a brand-new Version ID, but the same ETag as the original, an ETag is a content fingerprint, so a matching ETag proves the content is identical, byte for byte.

## Availability: Is the Data Still Reachable If Something Fails?

Availability means data stays reachable even if something breaks. This pillar is harder to *test* directly in a small lab, simulating a real data center outage isn't realistic from a free-tier account. Instead, I verified the configuration: my bucket used the **S3 Standard** storage class, which automatically distributes data across multiple Availability Zones by default, with no extra setup required. I'd already confirmed this in the JSON output from my earlier version checks, one piece of evidence serving two purposes.

Unlike the previous two sections, this one is more about confirming the configuration than attacking and recovering.

## Cleanup (Work in Progress)

Deleting the bucket turned out to be its own lesson. With versioning enabled, even `aws s3 rb --force` isn't enough:

```
aws s3 rb s3://test2-security-s3 --force
```

```
remove_bucket failed: BucketNotEmpty, You must delete all versions in the bucket.
```

The `--force` flag deletes current versions but leaves delete markers and older versions behind, nothing is ever silently lost, which is the whole point of versioning, but it does mean cleanup takes deliberate extra steps. I'm working through doing this properly via CLI (rather than just clicking through the console), that write-up is coming soon.

**If you've solved this before, I'd love to hear how.** My specific blocker: batch-deleting all versions and delete markers via `aws s3api delete-objects` requires passing JSON through `--delete`, and on Windows PowerShell I kept hitting encoding issues (UTF-16 line breaks, then a BOM at the start of the file) that made AWS's JSON parser reject the file. If you've run into this on PowerShell and found a clean fix, drop a comment or open an issue, genuinely curious what I'm missing.

## Lessons Learned

The biggest lesson wasn't any single command, it was how often I mixed up two similar-sounding things: Network ACLs vs. S3 ACLs, Version ID vs. Owner ID, an ARN's service field vs. its resource field. These weren't knowledge gaps so much as precision gaps, I understood the concepts, but hadn't yet built the habit of keeping the details straight under pressure.

That's really the value of building this hands-on instead of just reading about it. Misconfigurations are usually simple, once you see them clearly, but you only really see them clearly after tripping over them yourself.
