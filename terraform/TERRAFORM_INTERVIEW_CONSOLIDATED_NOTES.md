# Terraform Interview Consolidated Notes

## Core Concepts to Know

- Provider
- Resource
- Data source
- Variable
- Output
- Locals
- Module
- State
- Remote backend
- State locking
- Plan
- Apply
- Destroy
- Import
- Refresh / drift
- Lifecycle
- depends_on
- count
- for_each
- Dynamic blocks
- Workspaces
- Environment separation
- Secrets

---

## Enterprise Terraform Flow

```text
Git PR
  ↓
terraform fmt / validate
  ↓
Security scan
  ↓
terraform plan
  ↓
Review
  ↓
Approval
  ↓
Controlled apply
  ↓
Remote backend
  ↓
State locking
  ↓
RBAC / Audit
```

Example Azure backend:

```text
Storage Account
   |
Blob Container
   |
Terraform State
   |
RBAC
```

---

# Interview Q&A

## Q1. What happens if a resource was manually deleted but it still exists in Terraform code and state?

Terraform compares configuration, state, and the real infrastructure during planning.

If the resource is missing in the real environment but still exists in code, Terraform detects the drift and normally plans to recreate the resource.

```text
Code:   Resource exists
State:  Resource exists
Cloud:  Resource missing

terraform plan
       ↓
Terraform detects drift
       ↓
Plan: recreate resource
```

---

## Q2. What happens if a VM was manually resized but Terraform code and state still contain the old size?

Terraform detects the drift.

If the Terraform configuration still specifies the original size, the next apply normally changes the VM back to the value defined in code.

```text
Terraform code: Standard_D4
Cloud manually changed: Standard_D8

terraform plan
       ↓
Detects difference
       ↓
terraform apply
       ↓
Returns resource to Standard_D4
```

Unless the attribute is intentionally ignored through lifecycle configuration.

---

## Q3. What happens if a resource was manually deleted, removed from Terraform code, but still exists in state?

Terraform refreshes the state during planning.

Because the resource no longer exists remotely and no longer exists in configuration, Terraform reconciles that difference and removes the stale state reference as part of the normal planning/apply process.

The key point is to always run:

```bash
terraform plan
```

and inspect the proposed state/infrastructure changes before apply.

---

# State Locking Q&A

## Q4. What happens if the Terraform state is locked and an urgent production change must be deployed?

First determine whether the state lock is:

1. A legitimate active lock from another Terraform run
2. A stale lock left behind after a failed/crashed run

Flow:

```text
Urgent Production Change
        |
        v
Check why state is locked
        |
   +----+----------------------+
   |                           |
Active Terraform run       Stale lock
still running              previous run crashed
   |                           |
Wait / cancel safely           |
   |                           v
   |                    terraform force-unlock
   |                           |
   +-------------+-------------+
                 |
                 v
          terraform plan
                 |
          Verify carefully
                 |
                 v
          terraform apply
```

### If another legitimate apply is running

Do **not** force-unlock the state.

Either:

- Let the current run finish, or
- Cancel the existing run safely if the emergency requires it

Then verify the infrastructure and state before starting the emergency change.

Terraform can also wait for the lock:

```bash
terraform apply -lock-timeout=10m
```

### If the lock is stale

After confirming that no Terraform process is still using the state:

```bash
terraform force-unlock <LOCK_ID>
```

Then:

```bash
terraform plan
terraform apply
```

### If production cannot wait

For a genuine production outage, a minimal emergency manual change may sometimes be required.

```text
Production outage
      |
Terraform state locked
      |
Minimum safe manual mitigation
      |
Restore service
      |
terraform plan
      |
Detect / reconcile drift
      |
Bring code, state and infrastructure
back into alignment
```

Do not leave the manually changed infrastructure permanently outside Terraform management.

### Interview-ready answer

> “If Terraform state is locked during an urgent production change, I first determine whether it is an active or stale lock. I never force-unlock an active run because that can allow concurrent state writes. If the lock is stale, I verify no Terraform process owns it, use `terraform force-unlock <LOCK_ID>`, then run a fresh plan and apply. If there is a production outage that cannot wait, I make the minimum safe manual mitigation and immediately reconcile the resulting drift back into Terraform.”

---

## Q5. How do you get the Terraform Lock ID?

Terraform normally prints the Lock ID directly in the error when it fails to acquire the state lock.

Example:

```text
Error: Error acquiring the state lock

Lock Info:
  ID:        4d7f8c2a-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  Path:      terraform-prod/terraform.tfstate
  Operation: OperationTypeApply
  Who:       user@host
  Version:   1.x
  Created:   2026-10-05 12:30:00
```

The value shown after:

```text
ID:
```

is the lock ID.

Use it only after confirming the lock is stale:

```bash
terraform force-unlock 4d7f8c2a-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### Interview-ready answer

> “When Terraform cannot acquire the state lock, the error output includes the lock information and Lock ID. I take that ID and use it with `terraform force-unlock` only after verifying that no active Terraform run still owns the lock.”

---

## Q6. Why is force-unlocking an active state dangerous?

Two Terraform processes could then modify the same infrastructure and state concurrently.

Possible consequences:

- Conflicting infrastructure changes
- Incorrect state
- Lost state updates
- Resource recreation
- State corruption

Therefore:

```text
force-unlock
!= first response

force-unlock
= only after confirming stale lock
```

---

## Q7. How does Terraform Enterprise / HCP Terraform handle state locking?

Terraform Enterprise / HCP Terraform serializes runs for a workspace.

```text
Run A
  |
State / Workspace locked
  |
Run B
  |
Queued
```

For an emergency change:

```text
Check active run
      |
Cancel safely if appropriate
      |
Verify state / infrastructure
      |
Run emergency plan
      |
Review / approval
      |
Apply
```

The same principle remains:

> Never bypass an active lock without first understanding the run that owns it.
