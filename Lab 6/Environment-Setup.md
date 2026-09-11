# IKB42603 Lab 6 — Object Storage and Data Lifecycle

## Purpose

This report records the environment setup and task sequence for the lab **Object Storage and Data Lifecycle**. It is based on the supplied guide and the accompanying terminal-capture evidence. The lab uses LocalStack to emulate AWS services locally and the AWS CLI to administer an S3 bucket, IAM identity, and KMS key.

## Environment

| Component | Observed configuration |
| --- | --- |
| S3/KMS/IAM endpoint | `http://localhost:4566` (LocalStack) |
| Region | `us-east-1` |
| Credentials | AWS CLI test credentials (`test` / `test`) |
| Bucket used in the evidence | `miit-patient-records-25744` |
| IAM user | `DataAnalyst` |
| LocalStack image | `localstack/localstack:3.0` |
| Service ports | `4566:4566` and `4510-4559:4510-4559` |

## Step 1 — Start and configure the local AWS environment

1. Remove any previous LocalStack container and start the lab container:

   ```bash
   docker rm -f localstack 2>/dev/null
   docker run -d --name localstack \
     -p 4566:4566 \
     -p 4510-4559:4510-4559 \
     -v /var/run/docker.sock:/var/run/docker.sock \
     localstack/localstack:3.0
   ```

2. Point the AWS CLI at the local endpoint and configure its test account:

   ```bash
   export EP='--endpoint-url=http://localhost:4566'
   aws configure set aws_access_key_id test
   aws configure set aws_secret_access_key test
   aws configure set region us-east-1
   aws $EP sts get-caller-identity
   ```

3. Confirm that STS returns account `000000000000` and the root ARN. This verifies that subsequent commands target LocalStack rather than a real AWS account.

Evidence: [container startup](evidence/setup%201.png) and [CLI/STS verification](evidence/setup%202.png).

![LocalStack container startup](evidence/setup%201.png)

![AWS CLI configuration and STS identity check](evidence/setup%202.png)

## Step 2 — Classify data before storage

1. Prepare the three sample files: a public notice, an internal roster, and a confidential patient record.
2. Upload each object to a prefix that reflects its category and attach a `classification` tag:

   ```bash
   aws $EP s3api put-object --bucket $BUCKET --key public/notice.txt \
     --body public-notice.txt --tagging 'classification=public'
   aws $EP s3api put-object --bucket $BUCKET --key internal/roster.txt \
     --body internal-roster.txt --tagging 'classification=internal'
   aws $EP s3api put-object --bucket $BUCKET --key confidential/record.txt \
     --body confidential-record.txt --tagging 'classification=confidential'
   ```

3. List the bucket and inspect `confidential/record.txt` tags. The evidence shows all three objects and the confidential tag.

Evidence: [Task 1](evidence/task%201%20Classify%20data%20before%20storing%20it.png).

![Task 1 — Classify data before storing it](evidence/task%201%20Classify%20data%20before%20storing%20it.png)

## Step 3 — Reproduce the public-read breach

1. Create and attach a bucket policy that allows everyone (`Principal: "*"`) `s3:GetObject` on `arn:aws:s3:::${BUCKET}/*`.
2. Retrieve the policy to confirm it was attached.
3. Request the confidential object directly using the LocalStack S3 URL:

   ```bash
   curl -s -o leaked.txt -w 'HTTP %{http_code}\n' \
     http://localhost:4566/$BUCKET/confidential/record.txt
   cat leaked.txt
   ```

4. The result was HTTP 200 and returned the confidential patient content, demonstrating the risk created by a broad public bucket policy.

Evidence: [Task 2](evidence/task%202%20Reproduce%20the%20archetypal%20breach.png).

![Task 2 — Reproduce the archetypal breach](evidence/task%202%20Reproduce%20the%20archetypal%20breach.png)

## Step 4 — Remediate with S3 Block Public Access

1. Remove the unsafe bucket policy:

   ```bash
   aws $EP s3api delete-bucket-policy --bucket $BUCKET
   ```

2. Enable all four bucket-level public-access safeguards:

   ```bash
   aws $EP s3api put-public-access-block --bucket $BUCKET \
     --public-access-block-configuration \
     BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
   ```

3. Verify the returned configuration. The capture shows every setting as `true`.
4. Reapply the unsafe public policy and test direct anonymous retrieval. This test still returned HTTP 200 in LocalStack; therefore, it is an environment-specific limitation/outcome and should not be treated as production proof that Block Public Access is enforcing the expected denial.

Evidence: [Task 3.1](evidence/task%203.1%20Remediate%20with%20Block%20Public%20Access.png) and [Task 3.2](evidence/task%203.2.png).

![Task 3.1 — Remediate with Block Public Access](evidence/task%203.1%20Remediate%20with%20Block%20Public%20Access.png)

![Task 3.2 — Least-privilege resource policy](evidence/task%203.2.png)

## Step 5 — Compare identity and resource policies

1. Replace the broad policy with a least-privilege bucket policy granting the LocalStack root principal only `s3:GetObject` for `internal/*`.
2. Create IAM user `DataAnalyst`, attach an identity policy that allows `s3:GetObject` and `s3:ListBucket`, then create an access key and configure an `analyst` CLI profile. Do not place the displayed access-key secret in this report or source control.
3. Attach a bucket policy that explicitly allows this analyst `GetObject` for `internal/*` and explicitly denies the analyst all S3 actions for `confidential/*`.
4. Test using `AWS_PROFILE=analyst`:

   ```bash
   aws $EP s3api get-object --bucket $BUCKET --key internal/roster.txt analyst-internal.txt
   aws $EP s3api get-object --bucket $BUCKET --key confidential/record.txt analyst-confidential.txt
   ```

5. The internal retrieval was successful. Despite the intended explicit deny, the capture also shows a successful confidential retrieval; record this as a failed/unsupported policy-enforcement test in this LocalStack exercise and validate the policy in AWS before relying on it.

Evidence: [Task 4.1](evidence/task%204.1%20Identity%20policy%20versus%20resource%20policy.png), [Task 4.2](evidence/task%204.2.png), and [Task 4.3](evidence/Task%204.3.png).

![Task 4.1 — Identity policy versus resource policy](evidence/task%204.1%20Identity%20policy%20versus%20resource%20policy.png)

![Task 4.2 — Create and configure the DataAnalyst user](evidence/task%204.2.png)

![Task 4.3 — Test analyst access](evidence/Task%204.3.png)

## Step 6 — Enable default encryption at rest (SSE-KMS)

1. Create `encryption.json` with default SSE-KMS encryption, `$KEY_ID`, and S3 Bucket Keys enabled:

   ```json
   {"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms","KMSMasterKeyID":"$KEY_ID"},"BucketKeyEnabled":true}]}
   ```

2. Apply and retrieve the bucket encryption configuration:

   ```bash
   aws $EP s3api put-bucket-encryption --bucket $BUCKET \
     --server-side-encryption-configuration file://encryption.json
   aws $EP s3api get-bucket-encryption --bucket $BUCKET
   ```

3. Upload `confidential/record-v2.txt` and use `head-object` to verify `aws:kms`, the KMS key ARN, and `BucketKeyEnabled: true`.

Evidence: [Task 5](evidence/task%205%20Default%20Encryption%20at%20Rest%20%28SSE-KMS%29.png).

![Task 5 — Default Encryption at Rest (SSE-KMS)](evidence/task%205%20Default%20Encryption%20at%20Rest%20%28SSE-KMS%29.png)

## Step 7 — Delegated access and transport protection

1. Generate a 60-second presigned URL for `internal/roster.txt` and test it immediately and after expiry:

   ```bash
   URL=$(aws $EP s3 presign s3://$BUCKET/internal/roster.txt --expires-in 60)
   curl -s -w 'HTTP %{http_code}\n' "$URL"
   sleep 65
   curl -o /dev/null -w 'after expiry: HTTP %{http_code}\n' "$URL"
   ```

2. Both captured requests returned HTTP `000`, so the evidence does not validate URL use or expiry. This commonly occurs when the presigned host/signature does not match the LocalStack endpoint; troubleshoot endpoint/addressing configuration before treating the test as complete.
3. Add a bucket-policy explicit deny for all S3 actions where `aws:SecureTransport` is `false`, then list objects to confirm the bucket remains accessible through the CLI endpoint.

Evidence: [Task 6.1](evidence/task%206.1%20%20Delegated%20access%20and%20the%20condition-key%20trap.png) and [Task 6.2](evidence/task%206.2.png).

![Task 6.1 — Delegated access and the condition-key trap](evidence/task%206.1%20%20Delegated%20access%20and%20the%20condition-key%20trap.png)

![Task 6.2 — Secure transport bucket policy](evidence/task%206.2.png)

## Step 8 — Versioning, delete markers, and data remanence

1. Enable and verify bucket versioning:

   ```bash
   aws $EP s3api put-bucket-versioning --bucket $BUCKET \
     --versioning-configuration Status=Enabled
   aws $EP s3api get-bucket-versioning --bucket $BUCKET
   ```

2. Create successive versions of the confidential record, then list versions. The evidence shows two data versions (48 and 43 bytes) plus the initial/null version.
3. Delete the object without a version ID. This creates a delete marker; a normal `get-object` returns `NoSuchKey` while historic versions remain.
4. Delete the marker by its version ID to restore access to the latest object, then use a version-specific delete to permanently remove the null version. List versions again to confirm that only the expected historic versions remain.

Evidence: [Task 7.1](evidence/task%207.1%20Versioning%2C%20Delete%20Markers%20%26%20Data%20Remanence.png), [Task 7.2](evidence/task%207.2.png), [Task 7.3](evidence/task%207.3.png), and [Task 7.4](evidence/task%207.4.png).

![Task 7.1 — Enable versioning and create versions](evidence/task%207.1%20Versioning%2C%20Delete%20Markers%20%26%20Data%20Remanence.png)

![Task 7.2 — Delete marker and restoration](evidence/task%207.2.png)

![Task 7.3 — Permanently delete a specific version](evidence/task%207.3.png)

![Task 7.4 — Verify remaining versions](evidence/task%207.4.png)

## Step 9 — Lifecycle management and cryptographic erasure

1. Create and apply a lifecycle configuration with two enabled rules:
   - `RetireConfidentialRecords`: applies to `confidential/`, expires current objects after 365 days, and removes noncurrent versions after 30 days.
   - `AbortIncompleteUploads`: aborts incomplete multipart uploads after 7 days.
2. Verify the lifecycle rule IDs and statuses using `get-bucket-lifecycle-configuration`.
3. Identify the KMS key, disable it, and schedule it for deletion with a seven-day pending window:

   ```bash
   KEY_ID=$(aws $EP kms list-keys --query 'Keys[0].KeyId' --output text)
   aws $EP kms disable-key --key-id $KEY_ID
   aws $EP kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7
   ```

4. Verify the KMS key state is `PendingDeletion`. Note that the subsequent LocalStack `get-object` capture still returned data despite the pending key deletion. In AWS, key disablement/deletion makes KMS-encrypted data unreadable; the observed LocalStack result must not be used to validate cryptographic erasure semantics.

Evidence: [Task 8.1](evidence/task%208.1%20Lifecycle%2C%20Retention%20%26%20Cryptographic%20Erasure.png), [Task 8.2](evidence/task%208.2.png), and [Task 8.3](evidence/task%208.3.png).

![Task 8.1 — Lifecycle, retention, and cryptographic erasure](evidence/task%208.1%20Lifecycle%2C%20Retention%20%26%20Cryptographic%20Erasure.png)

![Task 8.2 — Disable and schedule KMS key deletion](evidence/task%208.2.png)

![Task 8.3 — Post-erasure retrieval attempt](evidence/task%208.3.png)

## Final verification

The supplied final verification confirms:

- Block Public Access: all four controls are `true`.
- Versioning: `Enabled`.
- Default encryption: `aws:kms` with the configured key.
- Lifecycle: `RetireConfidentialRecords` and `AbortIncompleteUploads` are `Enabled`.
- KMS key: `PendingDeletion`.

Evidence: [verification command](evidence/verification%20command.png).

![Final verification command](evidence/verification%20command.png)

## Cleanup and teardown

1. Remove the bucket policy and delete all object versions and delete markers.
2. Delete the bucket, the `DataAnalyst` inline policy and IAM user, and the LocalStack Docker container.
3. Remove local temporary JSON and text files created during the lab.

The supplied cleanup capture shows the version/deletion responses and `localstack` container removal. Exercise care in a real AWS account: deleting object versions and scheduling KMS deletion can make data irrecoverable.

Evidence: [cleanup and teardown](evidence/cleanup%20and%20teardown.png).

![Cleanup and teardown](evidence/cleanup%20and%20teardown.png)
