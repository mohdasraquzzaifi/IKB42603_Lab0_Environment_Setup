# IKB42603 Lab 5 — Environment Setup and Execution Report

Course: IKB42603 Cloud Computing Security Essentials

Lab: IKB42603 Lab5 Monitoring Logging and Incident Detection

Student ID: 52215225226

Name: MUHAMMAD ASRA QUZZAIFI BIN MOHD RABI

## 1. Purpose

This lab creates and centralises authentication logs, makes the records tamper-evident, detects a correlated attack pattern, and preserves incident evidence. The simulated incident is a brute-force attempt against `admin`, followed by a successful login and a 500 MB data export from `203.0.113.9`.


### 2 Configure the AWS CLI endpoint and central log destination

Set the LocalStack endpoint and create the CloudWatch log group and stream:

```bash
EP='--endpoint-url=http://localhost:4566'
aws $EP --region us-east-1 logs create-log-group --log-group-name /ccse/app
aws $EP --region us-east-1 logs create-log-stream --log-group-name /ccse/app --log-stream-name auth
```

**Evidence:** <img width="817" height="86" alt="setup localstack" src="https://github.com/user-attachments/assets/b39d4880-2d35-4586-bc82-ea304754939e" />
 shows the endpoint variable and successful creation commands for `/ccse/app` and its `auth` stream.

## 3. Task 1 — Generate application logs

Create the authentication log containing normal activity and the simulated attack sequence:

```bash
cat > auth.log <<'EOF'
2025-03-01T09:00:01 LOGIN_OK    user=ahmad   ip=10.0.0.5
2025-03-01T09:01:10 LOGIN_FAIL  user=admin   ip=203.0.113.9
2025-03-01T09:01:12 LOGIN_FAIL  user=admin   ip=203.0.113.9
2025-03-01T09:01:15 LOGIN_FAIL  user=admin   ip=203.0.113.9
2025-03-01T09:01:18 LOGIN_FAIL  user=admin   ip=203.0.113.9
2025-03-01T09:01:22 LOGIN_OK    user=admin   ip=203.0.113.9
2025-03-01T09:01:40 EXPORT_DATA user=admin   ip=203.0.113.9 size=500MB
EOF
cat auth.log
```

**Observed result:** seven log entries were created: four failed administrator logins from `203.0.113.9`, then a successful administrator login and a 500 MB export from that same address.  
**Evidence:** <img width="580" height="332" alt="task 1- generate application logs" src="https://github.com/user-attachments/assets/cdb948a2-2930-43dc-9a8a-b9a6e8235252" />


## 4. Task 2 — Centralise logs in CloudWatch

Ship each local record to the LocalStack CloudWatch log stream, then retrieve the stored messages:

```bash
TS=$(date +%s000)
while IFS= read -r line; do
  aws $EP logs put-log-events --log-group-name /ccse/app --log-stream-name auth \
    --log-events timestamp=$TS,message="$line" >/dev/null
  TS=$((TS+1000))
done < auth.log

aws $EP logs get-log-events --log-group-name /ccse/app --log-stream-name auth \
  --query 'events[].message' --output text
```

**Observed result:** the CloudWatch read-back contains all seven original messages, demonstrating that the logs were centralised rather than left only on the application host.  
**Evidence:** <img width="921" height="232" alt="task 2- centralisa logs" src="https://github.com/user-attachments/assets/ad41c858-e771-4b5a-a999-b5142e9c9e01" />


## 5. Task 3 — Query security-relevant activity

Count failed logins grouped by the relevant fields:

```bash
grep LOGIN_FAIL auth.log | awk '{print $4, $5}' | sort | uniq -c
```

**Observed result:** `4 ip=203.0.113.9`. This identifies four failed login attempts associated with the suspicious IP address. A log is the durable `LOGIN_FAIL` record; an event would be a near-real-time alert triggered by the repeated failures.  
**Evidence:** <img width="622" height="61" alt="task 3- query for security" src="https://github.com/user-attachments/assets/4fbc0aef-d547-44b6-9f32-cde833a362be" />


## 6. Task 4 — Create tamper-evident, hash-chained logs

Each hash is calculated from the preceding hash and the current log line. Therefore, altering any line changes that line’s hash and all following hashes.

```bash
PREV=0
while IFS= read -r line; do
  PREV=$(printf '%s%s' "$PREV" "$line" | sha256sum | cut -d' ' -f1)
  printf '%s | %s\n' "$line" "$PREV"
done < auth.log > auth.chain
cat auth.chain

sed 's/500MB/5MB/' auth.log > auth.tampered
cat auth.tampered
```

To complete the guide’s verification requirement, recompute the tampered chain and compare its final hash against the final hash in `auth.chain`:

```bash
PREV=0
while IFS= read -r line; do
  PREV=$(printf '%s%s' "$PREV" "$line" | sha256sum | cut -d' ' -f1)
done < auth.tampered
echo "Tampered final hash: $PREV"
tail -n 1 auth.chain
```

**Observed result:** `auth.chain` contains one SHA-256 hash per record. The altered copy changes the export size from `500MB` to `5MB`; recomputation must yield a different final hash, proving the alteration. The supplied capture shows the chain creation and changed record, but not the final-hash comparison itself.  
**Evidence:** <img width="1050" height="592" alt="task 4- create tamper-proof,hash-chianed logs" src="https://github.com/user-attachments/assets/0742374d-82c1-4e70-b363-ad178ea35c60" />


## 7. Task 5 — Detect the incident by correlation

Correlate the failures, subsequent successful login, and export activity for the same source IP:

```bash
IP=203.0.113.9
FAILS=$(grep -c "LOGIN_FAIL.*$IP" auth.log)
SUCCESS=$(grep -c "LOGIN_OK.*$IP" auth.log)
EXPORT=$(grep -c "EXPORT_DATA.*$IP" auth.log)
echo "IP=$IP fails=$FAILS success=$SUCCESS export=$EXPORT"

if [ "$FAILS" -ge 3 ] && [ "$SUCCESS" -ge 1 ] && [ "$EXPORT" -ge 1 ]; then
  echo 'ALERT: probable brute-force -> compromise -> data exfiltration'
fi
```

**Observed result:** `IP=203.0.113.9 fails=4 success=1 export=1`, followed by `ALERT: probable brute-force -> compromise -> data exfiltration`. The alert is possible only by correlating multiple records; no individual line proves the entire attack path.  
**Evidence:** <img width="640" height="212" alt="task 5- detect the incident" src="https://github.com/user-attachments/assets/1c52752f-c156-4fc5-a3fc-c71583c5fa49" />

## 8. Task 6 — Contain and preserve evidence

Model containment by blocking the suspicious IP, then preserve a timestamped evidence copy and its SHA-256 hash:

```bash
docker run --rm --cap-add=NET_ADMIN alpine sh -c \
  'apk add -q iptables; iptables -A INPUT -s 203.0.113.9 -j DROP; iptables -L INPUT -n | tail -2'

cp auth.log evidence_$(date +%Y%m%d).log
sha256sum evidence_*.log > evidence.sha256
cat evidence.sha256
```

**Observed result:** the containment output lists a `DROP` rule for source `203.0.113.9`. An evidence file named `evidence_20260902.log` was created and its SHA-256 value recorded in `evidence.sha256` as `0adc5d2ac06cbbdd366099bcc0540c4c0f76946e71b52e4c99322731696a203b`.  
**Evidence:** <img width="920" height="190" alt="task 6 - incident response and evidence preservation" src="https://github.com/user-attachments/assets/3ec51668-aca4-4a83-9829-0049086061b8" />


Verify the evidence has not changed:

```bash
sha256sum -c evidence.sha256
```

## 9. Incident summary

| Stage | Finding / action |
| --- | --- |
| Detection | The correlation rule detected four failed administrator logins, a successful login, and a data export from `203.0.113.9`. |
| Analysis | The sequence is consistent with brute-force access followed by potential data exfiltration. |
| Containment | A simulated `iptables` `DROP` rule blocked traffic from `203.0.113.9`. |
| Evidence and integrity | The original log was copied to a dated evidence file and hashed. A hash chain makes later log edits evident. |
| Lesson learned | Centralised, tamper-evident logs and correlation are required to turn isolated activity into actionable incident detection and defensible evidence. |

## 10. Final verification

```bash
aws --endpoint-url=http://localhost:4566 logs describe-log-groups
sha256sum -c evidence.sha256
```

Expected outcome: the `/ccse/app` log group is present and the evidence checksum reports `OK`.
