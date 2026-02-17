# Running Ultima Genomics WGS Variant Calling on AWS HealthOmics: A Production War Story

> **TL;DR:** Direct EC2 execution of Ultima split containers doesn't work at scale. This post documents every failure mode we hit processing 100–200+ GB CRAM files, and the exact configuration that finally made the workflow reliable and repeatable on AWS HealthOmics.

---

## Background

Ultima Genomics WGS data is not your typical sequencing workload. The combination of extremely large CRAM files, a split-container DeepVariant implementation, and tight coupling between containers and vendor-supplied workflow definitions creates a stack of compounding challenges that only reveal themselves at production scale.

Early experiments with small datasets tend to succeed — and that's the trap. The assumptions that hold at 10 GB collapse spectacularly once inputs exceed 100 GB and workflows must run unattended overnight.

This post captures the architectural decisions, concrete failure modes, and exact fixes required to get this right.

---

## Why Direct EC2 Execution Failed

The initial approach was straightforward: pull the Ultima containers, run them on EC2, done. This failed immediately and in non-obvious ways.

Ultima distributes DeepVariant as multiple split containers — `make_examples`, `call_variants`, and downstream QC/reporting live in separate images. This is fine when orchestrated by the vendor WDL, where internal paths, environment variables, and task dependencies are explicitly wired together. It is not fine when you try to invoke them directly.

The failure modes:
- Expected binaries were absent at documented paths (e.g., `/opt/deepvariant/bin/make_examples` did not exist)
- Executables were not exposed through `$PATH`
- Entrypoint overrides produced unstable, inconsistent behavior
- CLI behavior diverged from standard DeepVariant documentation

These were not user error. The containers encode implicit execution assumptions that only make sense inside the vendor WDL orchestration layer. Trying to reverse-engineer those assumptions outside a workflow engine was an architectural mismatch — not a debugging problem.

**Lesson:** Ultima split containers are not designed for standalone invocation. Full stop.

---

## The Fix: AWS HealthOmics Private Workflows

Migrating to AWS HealthOmics private workflows using the vendor-supplied WDL resolved the architectural mismatch immediately. Execution follows the workflow's native task structure, removing the need to manually manage container entrypoints or reverse-engineer internal paths.

HealthOmics also provides:
- Immutable workflow versions (critical for reproducibility)
- Managed scaling
- Native support for very large inputs via dynamic storage

---

## Automation Strategy

Once the execution model was correct, we built an automation layer to make sample submission reliable and repeatable. The strategy follows four steps:

```
LOOKUP → GENERATE → VALIDATE → TRIGGER
```

| Step | What Happens |
|------|-------------|
| **Lookup** | Fetch readset metadata from the HealthOmics Sequence Store |
| **Generate** | Create a `run_inputs.json` per sample |
| **Validate** | Check S3 for valid CRAM/CRAI pairs before submission |
| **Trigger** | Fire `aws omics start-run` |

This pattern catches ingestion problems early (before burning compute hours) and makes per-sample resubmission trivial:

```bash
./submit_readset_run.sh <READSET_ID>
./submit_readset_run.sh <READSET_ID> <CUSTOM_SAMPLE_NAME>
```

---

## Step-by-Step Configuration

### 1. Mirror Workflow Images into Amazon ECR

HealthOmics private workflows cannot pull from public registries. Every workflow image must be mirrored into Amazon ECR first.

```bash
REGION=us-east-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_BASE="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"

aws ecr get-login-password --region "$REGION" \
  | docker login --username AWS --password-stdin "$ECR_BASE"

aws ecr create-repository \
  --repository-name ultimagenomics-make_examples \
  --region "$REGION" >/dev/null 2>&1 || true

docker pull ultimagenomics/make_examples:3.1.10
docker tag  ultimagenomics/make_examples:3.1.10 \
            "$ECR_BASE/ultimagenomics-make_examples:3.1.10"
docker push "$ECR_BASE/ultimagenomics-make_examples:3.1.10"
```

Repeat for all images in the workflow (`call_variants`, QC, etc.).

---

### 2. Grant HealthOmics Permission to Pull Images

Mirroring images is not enough. HealthOmics requires an explicit ECR repository policy, or runs will fail at runtime with `ECR_PERMISSION_ERROR`.

```bash
cat > /tmp/omics-ecr-policy.json <<'JSON'
{
  "Version": "2008-10-17",
  "Statement": [
    {
      "Sid": "AllowOmicsPull",
      "Effect": "Allow",
      "Principal": { "Service": "omics.amazonaws.com" },
      "Action": [
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchCheckLayerAvailability"
      ]
    }
  ]
}
JSON

aws ecr set-repository-policy \
  --region us-east-1 \
  --registry-id <YOUR_ACCOUNT_ID> \
  --repository-name ultimagenomics-make_examples \
  --policy-text file:///tmp/omics-ecr-policy.json
```

Apply this policy to every mirrored repository.

---

### 3. Update Image References in the WDL

All public image URIs in the vendor WDL must be replaced with ECR URIs. Do this programmatically:

```bash
cd workflows/efficient_dv
REGION=us-east-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_BASE="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"

find . -type f -name "*.wdl" -print0 | xargs -0 sed -i.bak \
  -e "s|ultimagenomics/make_examples:3.1.10|$ECR_BASE/ultimagenomics-make_examples:3.1.10|g" \
  -e "s|ultimagenomics/call_variants:2.2.4|$ECR_BASE/ultimagenomics-call_variants:2.2.4|g"

find . -type f -name "*.bak" -delete
```

Then package and publish a new immutable workflow version:

```bash
zip -r efficient_dv_bundle.zip . -x "*.git*" "__pycache__/*"

aws omics create-workflow-version \
  --region us-east-1 \
  --workflow-id <WORKFLOW_ID> \
  --definition-zip fileb://efficient_dv_bundle.zip \
  --version-name fix-qcreport-cpu2-resource-hardened
```

---

### 4. Validate Input Schema Before Submission

HealthOmics enforces strict input validation against the workflow's `parameterTemplate`. Enumerate required parameters before attempting a run:

```bash
aws omics get-workflow \
  --region us-east-1 \
  --id <WORKFLOW_ID> \
  --query 'parameterTemplate' \
  --output json > parameter_template.json

jq -r 'to_entries[] | select(.value.optional==false) | .key' parameter_template.json
```

This surfaces missing required fields before they cause cryptic runtime failures.

---

### 5. Resource Hardening for Large CRAMs

This was the most painful lesson. A run progressed for ~3.5 hours and then failed during QC reporting with a **NumPy out-of-memory error**. QC tasks scale aggressively with input size and must be treated as heavyweight compute stages — not afterthoughts.

Apply runtime overrides at submission time:

```bash
jq '
  with_entries(.key |= sub("^EfficientDV\\."; "")) |
  .ug_make_examples_memory_override = 16 |
  .ug_make_examples_cpus_override   = 4  |
  .ug_call_variants_extra_mem       = 16 |
  .call_variants_cpus               = 12 |
  .call_variants_threads            = 12
' run_inputs.json > run_inputs.tmp && mv run_inputs.tmp run_inputs.json
```

The `QCReport` task should be provisioned with **64 GB RAM and 2 CPUs**. This converted consistent late-stage failures into consistent completions.

---

### 6. Submitting Runs

```bash
aws omics start-run \
  --region us-east-1 \
  --workflow-id <WORKFLOW_ID> \
  --workflow-version-name fix-qcreport-cpu2-resource-hardened \
  --role-arn <HEALTHOMICS_ROLE_ARN> \
  --name example-run \
  --output-uri s3://<OUTPUT_BUCKET>/ \
  --parameters file://run_inputs.json
```

---

### 7. Debugging Failed Runs

When a run fails, retrieve task metadata and logs:

```bash
# Get task-level metadata
aws omics get-run-task \
  --region us-east-1 \
  --id <RUN_ID> \
  --task-id <TASK_ID> \
  --output json

# Tail CloudWatch logs for the failed task
aws logs get-log-events \
  --region us-east-1 \
  --log-group-name /aws/omics/WorkflowLog \
  --log-stream-name run/<RUN_ID>/task/<TASK_ID> \
  --limit 200 \
  --query 'events[].message' \
  --output text
```

---

## Storage and Ingestion

For large CRAMs, the workflow uses `storageType = DYNAMIC`, allowing disk resources to scale with input size and preventing mid-run disk exhaustion.

CRAMs must be ingested into the HealthOmics Sequence Store as valid CRAM/CRAI pairs. Ingestion failures citing `Incorrect source file provided` were resolved by validating file integrity with `samtools` prior to import:

```bash
samtools quickcheck sample.cram
samtools view -H sample.cram > /dev/null  # validates index lookup
```

Automate this check in your submission script before firing `aws omics start-run`.

---

## Summary of Failure Modes and Fixes

| Failure | Root Cause | Fix |
|---------|-----------|-----|
| Missing binaries / bad paths on EC2 | Containers designed for WDL orchestration, not direct invocation | Migrate to HealthOmics private workflows |
| `ECR_PERMISSION_ERROR` at runtime | No ECR repository policy for HealthOmics service principal | Add `omics.amazonaws.com` pull policy to each ECR repo |
| Input validation errors | Missing required parameters | Enumerate `parameterTemplate` before submission |
| OOM crash after 3.5 hours | QC task under-provisioned for large inputs | Set QCReport to 64 GB RAM; apply memory/CPU overrides |
| CRAM ingestion failures | Invalid CRAM/CRAI pairs in Sequence Store | Validate with `samtools` before import |

---

## Conclusion

AWS HealthOmics is a stable, scalable platform for large-scale Ultima WGS variant calling — but only once the right constraints are addressed:

- **Do not** attempt direct EC2 invocation of Ultima split containers
- **Always** mirror images to ECR and grant HealthOmics pull permissions
- **Treat QC stages as compute-heavy** and provision accordingly
- **Validate inputs early** to fail fast rather than late

Once these are in place, the workflow becomes predictable, reproducible, and straightforward to operate at scale.

---

*Workflow version used: `fix-qcreport-cpu2-resource-hardened` | Storage: Dynamic | Tested on CRAM inputs 100–200+ GB*
