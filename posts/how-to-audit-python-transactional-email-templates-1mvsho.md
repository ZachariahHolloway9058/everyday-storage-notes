# How to Audit Python Transactional Email Templates — 3 Welcome Suppression List Checks

An edtech compliance notice needs an explainable record, but collecting every available event creates integration work without necessarily proving that a learner received anything. Keep a small, explicit evidence contract: preserve the template revision, record the suppression decision, and attach delivery observations without upgrading their meaning. Short answer: compare transactional email integrations by the effort required to produce those three records for a welcome message containing a compliance notice; leave unit-price comparisons until that requirement is satisfied.

Evidence needs boundaries.

The following acceptance exercise uses Python and a local SQLite database to make that contract executable. All learner identifiers, revisions, and events are synthetic test inputs. This is an application design, not a claim about any provider's features, a legal retention requirement, or a production durability configuration. Its editorial angle is evidence reconstruction: can a reviewer explain one notice from the stored record without consulting the current template or guessing what a status meant?

## How should welcome email templates and a suppression list support an audit?

Start by writing the claim the application is allowed to make. A notice can be prepared, withheld by a suppression rule, submitted to a transport, or associated with a subsequent transport observation. Keep those statements distinct. For this exercise, an observation called accepted means only that the test transport accepted the submission; it does not establish delivery, reading, comprehension, or compliance. A useful record must retain the origin and meaning of a delivery observation before anyone promotes it into an institutional assertion.

The integration question then becomes concrete: how much code and operational procedure will the team own to connect a rendered template, an eligibility decision, and an externally supplied event? A short send function can conceal a large reconciliation obligation, particularly if the dashboard exposes a current status while the institution needs the sequence of decisions behind that status. Evaluate that sequence with synthetic records before choosing where templates or suppressions should live. Consider the synthetic pair used below: learner-014 is withheld and learner-015 is eligible, yet both notices use welcome-r3. A reviewer examining only the template table cannot explain the difference, while a reviewer examining only the final delivery status cannot establish which policy revision authorized submission. Preserve the join between those records, then ask an engineer unfamiliar with the integration to reconstruct both outcomes from the evidence view. If that exercise requires access to a mutable dashboard or an undocumented interpretation of a status, the integration estimate is missing work.

Don't infer receipt.

For a welcome email that also contains a compliance notice, decide which content may share the same delivery policy. In this design, a suppressed recipient produces an explicit withheld record and an institutional review item. It never silently triggers a different sending identity or an SMS fallback. The team responsible for the notice must specify the alternate process and its evidence requirements; an email integration cannot supply that policy by implication.

## Step 1: Freeze the rendered template before submission

Run the Python blocks in order in the same interpreter. They use the standard library and an in-memory database so that the acceptance exercise leaves no persistent learner data behind. The first check binds a fictional notice to a template revision and rendered text. A production design should also specify how it preserves the actual submitted payload, including transport-added fields that this example does not model.

```python
import hashlib
import json
import sqlite3

db = sqlite3.connect(":memory:")
db.executescript("""
CREATE TABLE notice (
    notice_id TEXT PRIMARY KEY,
    learner_id TEXT NOT NULL,
    template_revision TEXT NOT NULL,
    rendered_body TEXT NOT NULL,
    body_sha256 TEXT NOT NULL
);
CREATE TABLE decision (
    notice_id TEXT PRIMARY KEY REFERENCES notice(notice_id),
    disposition TEXT NOT NULL,
    policy_revision TEXT NOT NULL
);
CREATE TABLE observation (
    source TEXT NOT NULL,
    event_id TEXT NOT NULL,
    notice_id TEXT NOT NULL REFERENCES notice(notice_id),
    kind TEXT NOT NULL,
    payload_json TEXT NOT NULL,
    PRIMARY KEY (source, event_id)
);
""")
db.execute("PRAGMA foreign_keys = ON")

body = "Welcome. Review the course compliance notice in your learner portal."
digest = hashlib.sha256(body.encode("utf-8")).hexdigest()
with db:
    db.execute(
        "INSERT INTO notice VALUES (?, ?, ?, ?, ?)",
        ("notice-014", "learner-014", "welcome-r3", body, digest),
    )

stored_body, stored_digest = db.execute(
    "SELECT rendered_body, body_sha256 FROM notice WHERE notice_id = ?",
    ("notice-014",),
).fetchone()
assert hashlib.sha256(stored_body.encode("utf-8")).hexdigest() == stored_digest
```

The assertion checks that the saved text matches its saved digest. That is a narrow integrity check. Someone able to replace both fields can defeat it, so a hash in the same database is insufficient evidence of tamper resistance. Specify access controls, backup recovery, and the institution's required retention process separately, and test a restore before treating the storage layer as an archive. The in-memory database above is deliberately disposable.

Retaining rendered content makes reconstruction easier but also retains whatever learner information the rendering contains. Choose the minimum evidence content your review process actually requires, document who can read it, and assign deletion rules to both the message record and its backups. A template revision alone is a smaller record; it also leaves reconstruction dependent on the template and input data remaining available. That is the trade-off to sign off, rather than assuming a revision string preserves the message.

## Step 2: Make the suppression decision inspectable

Use a local policy fixture to test the withheld branch. This set represents an application decision; it does not describe a provider suppression API, and the demonstration does not establish legal eligibility to send a notice. Store the policy revision alongside the outcome so a later policy edit cannot silently reinterpret the recorded decision.

```python
suppressed_learners = {"learner-014"}
policy_revision = "notice-policy-r2"

def record_decision(notice_id):
    row = db.execute(
        "SELECT learner_id FROM notice WHERE notice_id = ?",
        (notice_id,),
    ).fetchone()
    if row is None:
        raise ValueError("Unknown notice")
    disposition = "withheld" if row[0] in suppressed_learners else "eligible"
    with db:
        db.execute(
            "INSERT INTO decision VALUES (?, ?, ?)",
            (notice_id, disposition, policy_revision),
        )
    return disposition

assert record_decision("notice-014") == "withheld"
```

The pass condition is a withheld record with its policy revision and no transport submission.

In a production implementation, also store the decision time and the specific reason or evidence reference needed by the authorized reviewer. This compact fixture uses one suppression reason; it should not collapse complaints, invalid addresses, and preference changes into an unexplained boolean in the eventual integration.

Name the race: eligibility can change between preparation and submission. Define where the final eligibility check occurs and how concurrent policy updates interact with dispatch. The example intentionally has no transport call, queue, or concurrency guarantee. Adding a queue requires an explicit handoff design, including what the worker does when it sees an older policy revision; the saved decision remains evidence of that earlier decision, not permission to ignore a newer restriction.

Unsubscribe handling needs its own boundary. RFC 8058 specifies one-click unsubscribe signaling for list email, including List-Unsubscribe and List-Unsubscribe-Post headers and a POST interaction. It does not decide whether this institution's welcome notice belongs in that category, nor does it replace an application suppression policy. Apply its requirements where the message category calls for them, and have the policy owner decide how optional welcome content and required notices should be separated.

## Step 3: Reconcile observations without inventing receipt

Now add a second synthetic notice that is eligible under the same fixture. The adapter below takes normalized test events with a source identifier, event identifier, notice identifier, and kind. These fields are an application contract, not a promised webhook schema. Real adapters must authenticate incoming events, validate their shape, and map external message identifiers to local notice identifiers before calling this function.

```python
with db:
    db.execute(
        "INSERT INTO notice VALUES (?, ?, ?, ?, ?)",
        ("notice-015", "learner-015", "welcome-r3", body, digest),
    )
assert record_decision("notice-015") == "eligible"

def record_observation(event):
    encoded = json.dumps(event, sort_keys=True, separators=(",", ":"))
    with db:
        previous = db.execute(
            "SELECT payload_json FROM observation WHERE source = ? AND event_id = ?",
            (event["source"], event["event_id"]),
        ).fetchone()
        if previous is not None:
            if previous[0] != encoded:
                raise ValueError("Event identity reused with different content")
            return False
        decision = db.execute(
            "SELECT disposition FROM decision WHERE notice_id = ?",
            (event["notice_id"],),
        ).fetchone()
        if decision != ("eligible",):
            raise ValueError("Observation needs reconciliation against dispatch evidence")
        db.execute(
            "INSERT INTO observation VALUES (?, ?, ?, ?, ?)",
            (event["source"], event["event_id"], event["notice_id"],
             event["kind"], encoded),
        )
    return True

event = {
    "source": "test-transport",
    "event_id": "event-015",
    "notice_id": "notice-015",
    "kind": "accepted",
}
assert record_observation(event) is True
assert record_observation(event) is False
assert db.execute("SELECT COUNT(*) FROM observation").fetchone()[0] == 1
```

This fixture checks sequential duplicate handling. It does not implement concurrent ingestion or prove that a message was submitted: the eligible check is only a local consistency guard. In a deployed service, link observations to a recorded dispatch attempt, preserve anomalous events in a restricted reconciliation store, and alert on the mismatch rather than discarding the evidence. Also record source event time and local receipt time separately so an arrival sequence cannot masquerade as the transport's event sequence.

A second failure mode is the ambiguous submission: a worker loses its response and cannot tell from the local record whether another attempt would duplicate the notice. Make the recovery decision part of the adapter acceptance exercise, using the actual transport's documented identifiers and retry semantics. Do not label a fresh application identifier an exactly-once guarantee. The duplicate-event check above addresses repeated observations, which is a different problem.

## Compare the evidence work, then roll out one notice type

The comparison belongs here, after the required records are explicit. Use the same synthetic notices to compare each proposed API integration, and estimate implementation and ongoing review work against the same acceptance criteria. A shortlist containing Amazon SES, Resend, Postmark, or Mailgun still needs this evidence exercise: the names alone establish nothing about the institution's ability to reconstruct a notice. This compares placements of responsibility; it makes no claim about a particular product's capabilities.

| Design choice | Work the application retains | Acceptance question |
| --- | --- | --- |
| Render templates inside the application | Revision control, rendering, payload retention | Can a reviewer recover the exact submitted content after a template edit? |
| Render templates through an external interface | Revision mapping and evidence capture across that interface | What record binds the rendered output to this notice? |
| Keep an application suppression ledger | Policy updates, final eligibility checks, reconciliation | Can a withheld notice be explained without sending it again? |
| Depend on external suppression decisions | Importing reasons and preserving their meaning locally | Can the decision still be reconstructed after external records expire? |
| Preserve individual observations | Deduplication, event mapping, access and retention controls | Can duplicate or delayed events change the claimed outcome incorrectly? |

Count engineering work across template changes, suppression updates, event verification, recovery drills, and review tooling. Storage and retention have operational costs too. Obtain current commercial terms only after these boundaries are understood; a unit price cannot tell you how many adapters, exception queues, or manual investigations the institution will own.

For rollout, start with one notice type and synthetic recipients. Require the three checks to pass, then test a template change, an eligibility change before dispatch, a repeated event, and an event whose identity is reused with different content. Verify that the review view distinguishes withheld, submitted, and observed states, and that a restricted operator can recover the required evidence from a backup. Deploy the event receiver before enabling real submissions so the first observations have somewhere to go.

Expand only after reconciliation is owned.

Watch the age of unresolved dispatch attempts, unmatched observations, and withheld notices awaiting institutional review; assign an operator and an escalation rule to each queue. The release criterion is a reconstructable notice record with explicit limits on what it proves.

## References

RFC 8058 supplies the one-click unsubscribe boundary discussed above. The SQLite references document the local demonstration's transaction, constraint, and foreign-key mechanisms; they do not establish production retention or compliance guarantees.

## Sources

- https://datatracker.ietf.org/doc/html/rfc8058
- https://docs.python.org/3/library/sqlite3.html
- https://www.sqlite.org/foreignkeys.html
- https://www.sqlite.org/lang_createtable.html
