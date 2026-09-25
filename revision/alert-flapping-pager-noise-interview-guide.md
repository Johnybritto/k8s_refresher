# Alert Flapping and Pager Noise — Interview Guide

## Scenario

You are paged for alerts, but many of them auto-resolve shortly afterward. Teams are repeatedly getting paged and there is a lot of noise.

This is commonly called:

- alert flapping
- noisy alerting
- low signal-to-noise alerting

The issue is usually not that alerts auto-resolve. Auto-resolution is useful.

The real problem is that transient or low-value conditions are triggering pages too easily.

---

# 1. First Principle

> Every page should be actionable. If no human action is required, it should probably not be a page.

The goal is not simply to reduce alert count.

The goal is to improve **signal-to-noise ratio** so that when the on-call engineer is paged, the alert is meaningful and needs action.

---

# 2. Investigation Flow

Before changing thresholds, identify why alerts are firing.

~~~text
Alert fires
    |
    v
Is there real user impact?
    |
    +--> Yes --> keep as actionable paging candidate
    |
    +--> No
          |
          v
Is it transient?
          |
          +--> Yes --> add persistence / hysteresis / lower severity
          |
          +--> No --> inspect alert logic and dependency chain
~~~

Correlate the alert with:

- latency
- error rate
- availability
- request failures
- application health
- host/pod health
- business transactions
- downstream dependencies

---

# 3. Add a Persistence Window

Bad alert:

~~~text
CPU > 80%
   |
   v
PAGE immediately
~~~

A short spike can create a page even if the system recovers naturally.

Better:

~~~text
CPU > 80%
continuously for 5–10 minutes
       |
       v
      ALERT
~~~

This is commonly implemented using a duration or persistence condition.

For example, in Prometheus this is the `for:` duration.

---

# 4. Use Different Trigger and Recovery Thresholds

If an alert fires and resolves around the same threshold, it can flap repeatedly.

Bad:

~~~text
Alert when CPU > 80%
Resolve when CPU < 80%
~~~

If CPU moves between 79% and 81%:

~~~text
81% -> ALERT
79% -> RESOLVE
81% -> ALERT
79% -> RESOLVE
~~~

Better:

~~~text
Alert when CPU > 85%

Resolve only when CPU < 70%
~~~

This gap is called **hysteresis**.

It prevents repeated open/close cycles near the threshold.

---

# 5. Add a Recovery Stability Window

Sometimes an alert is unhealthy for several minutes, becomes healthy for only a few seconds, resolves, and then immediately fires again.

Instead:

~~~text
Alert condition clears
       |
       v
Remain healthy for 5 minutes
       |
       v
Resolve incident
~~~

This reduces:

~~~text
OPEN
CLOSE
OPEN
CLOSE
OPEN
~~~

behavior.

---

# 6. Page Only Actionable Severity Levels

Do not send every warning to the pager.

A practical model:

~~~text
P1 / Critical
   -> Pager / Phone / On-call

P2
   -> Teams/Slack + Ticket

P3 / Warning
   -> Dashboard / Email / Trend monitoring
~~~

Example:

~~~text
CPU > 80%
    -> Warning

CPU > 95%
AND
application latency/errors impacted
for 10 minutes
    -> Page
~~~

The exact thresholds depend on the application and capacity model.

---

# 7. Prefer Symptom-Based Alerts Over Raw Resource Metrics

Low-level metrics are useful for diagnosis, but many should not page directly.

Potentially noisy signals:

- CPU spike
- short memory spike
- one pod restart
- one process restart
- temporary queue growth
- temporary node pressure

Better paging signals:

- service unavailable
- sustained high error rate
- latency SLO breach
- failed customer transactions
- quorum loss
- replication failure
- capacity exhaustion
- sustained dependency outage

Example:

~~~text
CPU high
   |
   +--> dashboard / warning

Error rate high + latency high + user impact
   |
   +--> PAGE
~~~

---

# 8. Group Related Alerts

One infrastructure failure can generate many dependent alerts.

Example:

~~~text
NodeDown
   |
   +-- PodDown x 20
   +-- ApplicationUnavailable x 10
   +-- ExporterDown
   +-- DiskUnavailable
~~~

Without grouping, this may create dozens of pages.

Instead:

~~~text
NodeDown
   |
   v
Create one primary incident
   |
   v
Group dependent symptoms
~~~

The on-call engineer sees the root cause rather than a flood of symptoms.

---

# 9. Deduplicate Repeated Alerts

If the same alert condition generates repeated notifications for the same incident, deduplicate it.

Conceptually:

~~~text
Same alert
Same resource
Same active incident
       |
       v
Do not create another page
~~~

Update the existing incident instead.

---

# 10. Use Alert Inhibition / Suppression

If a parent/root-cause alert is already active, suppress dependent alerts.

Example:

~~~text
HostDown
   |
   +-- suppress ProcessDown
   +-- suppress ExporterDown
   +-- suppress AppInstanceDown
   +-- suppress DiskAgentDown
~~~

This is often called:

- inhibition
- suppression
- event correlation
- dependency-based suppression

Prometheus Alertmanager provides inhibition rules; other monitoring products provide similar capabilities.

---

# 11. Build Dependency-Aware Alerting

Do not page multiple teams for the same root cause.

Example:

~~~text
Database Down
      |
      +--> API Errors
      +--> Application Errors
      +--> Queue Errors
~~~

If the database is the root cause:

~~~text
Primary Incident:
Database Down

Dependent alerts:
Grouped / Suppressed / Linked
~~~

This makes triage much faster.

---

# 12. Use SLO and Error-Budget-Based Alerting

For critical services, consider moving from pure static thresholds toward SLO-based alerting.

Example:

~~~text
Availability SLO = 99.9%
~~~

Instead of paging on:

~~~text
CPU > 85%
~~~

page when:

~~~text
High error-budget burn
        +
Sustained user impact
        |
        v
       PAGE
~~~

This aligns alerting with customer impact.

---

# 13. Example of a Noisy Alert

Current behavior:

~~~text
Memory > 80%
      |
      v
Pager fires
      |
2 minutes later
      |
Memory = 78%
      |
Auto-resolve
      |
Memory = 81%
      |
Pager fires again
~~~

This is classic alert flapping.

A better design may be:

~~~text
Memory > 90%
continuously for 10 minutes
          +
Application latency/error impact
          |
          v
         PAGE
~~~

Recovery:

~~~text
Memory < 75%
continuously for 5 minutes
         |
         v
       RESOLVE
~~~

The original 80% condition can remain as:

~~~text
Warning / dashboard signal
~~~

rather than an on-call page.

---

# 14. Alert Tuning Workflow

Use alert history instead of changing thresholds randomly.

~~~text
Export alert history
       |
       v
Identify top noisy alerts
       |
       v
Classify:
- actionable
- non-actionable
- transient
- duplicate
- dependent
       |
       v
Tune rules
       |
       v
Observe for a period
       |
       v
Measure page reduction and missed incidents
~~~

Useful metrics:

- pages per on-call shift
- percentage of auto-resolved alerts
- percentage requiring human action
- duplicate alert count
- mean time to acknowledge
- mean time to resolve
- false-positive rate
- after-hours page count

---

# 15. Do Not Simply Disable Auto-Resolve

Auto-resolve is useful.

It tells responders that the monitored condition has recovered.

The better solution is to improve:

- trigger logic
- persistence
- recovery logic
- severity
- grouping
- deduplication
- inhibition
- dependency correlation

---

# 16. Strong Interview Answer

> "If teams are being paged repeatedly and the alerts auto-resolve soon afterward, I would treat that as alert flapping and a signal-to-noise problem rather than simply disabling auto-resolution.
>
> First I would review alert history and determine whether the conditions represent genuine user impact or short-lived infrastructure fluctuations. For transient conditions I would add a persistence window so the alert must remain unhealthy for several minutes before firing.
>
> I would also use separate trigger and recovery thresholds to introduce hysteresis, and potentially require the system to remain healthy for a few minutes before resolving the incident.
>
> I would make sure only actionable P1/P2 conditions page the on-call team. Lower-level CPU, memory or restart signals can remain as warnings or diagnostic alerts unless they correlate with customer impact.
>
> I would then introduce grouping, deduplication and inhibition so one root-cause event does not generate dozens of dependent pages. For critical applications I would move toward symptom- and SLO-based alerts such as availability, error rate, latency and error-budget burn rather than paging on every raw infrastructure threshold.
>
> The objective is not to reduce alerts blindly. The objective is that when an engineer gets paged, the alert is meaningful, actionable and requires human intervention."

---

# 17. Follow-Up Interview Q&A

## Q1. What is alert flapping?

An alert repeatedly changes between firing and resolved states because the monitored value oscillates near the threshold or recovers only briefly.

---

## Q2. Is auto-resolution itself bad?

No.

Auto-resolution is useful.

The problem is when alerts fire too aggressively for short-lived conditions and repeatedly create incidents.

---

## Q3. What is the first thing you check?

Whether the alert corresponds to real application or customer impact.

Do not immediately change thresholds without understanding the signal.

---

## Q4. How do you prevent short spikes from paging?

Add a persistence duration.

Example:

~~~text
CPU > 90% for 10 minutes
~~~

instead of paging on a single sample.

---

## Q5. What is hysteresis?

Using different thresholds for triggering and recovery.

Example:

~~~text
Fire > 90%
Resolve < 70%
~~~

This prevents flapping around a single threshold.

---

## Q6. What if the alert clears for 20 seconds and fires again?

Add a recovery stability window.

For example:

~~~text
Condition healthy continuously for 5 minutes
    -> resolve
~~~

---

## Q7. Should CPU alerts page the team?

Not necessarily.

CPU is often better as a diagnostic or warning signal.

Page when high CPU is sustained and results in meaningful service impact, capacity exhaustion, or an actionable condition.

---

## Q8. What is alert inhibition?

Suppressing secondary alerts when a known root-cause alert is already active.

Example:

~~~text
NodeDown active
   -> suppress PodDown alerts caused by that node
~~~

---

## Q9. What is alert grouping?

Combining multiple related alerts into one incident or notification.

This reduces notification storms.

---

## Q10. What is deduplication?

Avoiding repeated pages for the same active condition/resource.

Update the existing incident instead of creating another one.

---

## Q11. What is the difference between warning and paging alerts?

A warning indicates something may need attention.

A paging alert means:

> A human must act now or soon to prevent or mitigate meaningful impact.

---

## Q12. What are better paging signals?

Examples:

- service availability
- sustained error rate
- latency
- transaction failure
- quorum loss
- replication failure
- capacity exhaustion
- high SLO burn rate

---

## Q13. Why use SLO-based alerts?

They align paging with service reliability and customer impact rather than isolated infrastructure metrics.

---

## Q14. Should we just increase all thresholds?

No.

That may hide real failures.

Tune alerts based on:

- historical behavior
- capacity
- user impact
- SLOs
- actionability

---

## Q15. How do you know whether tuning worked?

Measure:

- number of pages
- duplicate pages
- auto-resolved pages
- percentage requiring human action
- missed incidents
- false positives
- acknowledgement time
- resolution time

The goal is fewer low-value pages without reducing incident detection quality.

---

# 18. One-Line Memory Aid

> **Persistence + Hysteresis + Recovery Stability + Severity + Grouping + Deduplication + Inhibition + SLO-based paging = lower alert noise.**
