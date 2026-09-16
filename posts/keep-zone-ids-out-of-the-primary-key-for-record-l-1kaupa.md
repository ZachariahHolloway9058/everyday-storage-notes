# Keep Zone IDs Out of the Primary Key for Record Lookup (2 Failure Modes)

If you want the least complicated thing that survives a provider migration: give every custom domain a primary key your application generates, keep the provider's DNS zone id beside it as an indexed nullable column, and key record operations on the RRset triple — domain, name, type — instead of on whatever opaque string the DNS API handed back. A zone id describes your relationship with one provider account. It is not the identity of the seller's domain.

That is the whole recommendation. The interesting part is why the storage bill makes it obvious.

## Where the zone inventory bill actually lands

Take the inventory shape a marketplace ends up with once sellers can point their own domains at the product: 12,000 domains, each carrying an apex A or ALIAS, a www CNAME, an MX set, one SPF TXT, two DKIM selector TXTs, and a `_dmarc` TXT. Call it fifteen records per domain. That is 180,000 rows. On any relational engine, that is a few dozen megabytes with indexes, and you could keep every version of it since the company was founded without ever seeing the line item on a bill.

The money is somewhere else.

Two streams dominate, and neither of them is the zone table. The first is observation: if the reconciler polls six interesting RRsets per domain every fifteen minutes — a sane interval when sellers are editing DNS at their own registrar and you need to notice within a coffee break — you are writing 12,000 × 6 × 96 = 6,912,000 observation rows per day. The second is deliverability evidence. DMARC aggregate reports arrive as one XML document per reporting organization per domain per reporting interval, and the default interval in RFC 7489 is 86,400 seconds, so a domain heard from by eight reporters produces eight documents a day; across the fleet that's 96,000 documents daily, each one expanding into a row per source IP per authentication result. Inventory is 180,000 rows once. Evidence is seven million rows a day, forever, if you let forever happen.

Four orders of magnitude. Any primary-key argument that ignores that ratio is an argument about the smallest table in the system.

The change that moves the dominant term isn't a key choice at all — it's refusing to write rows that say nothing. Hash the canonical RRset, compare it against the last hash you stored for that triple, and journal only transitions. Seven million observations a day collapse into the few hundred changes that actually happened, and the retained artifact becomes a change history rather than a polling log. Which is precisely where identity starts to matter, because a change history is worthless if its foreign key dies the day a seller moves DNS providers.

## Should the application store the DNS zone id itself as a primary key for record lookup?

No, and not for reasons of schema purity. Zone ids have three properties that a primary key must not have, and a marketplace hits all three in the first year.

They are null at insert time. The domain row needs to exist the moment a seller types `shop.example.com` into a form, because that is when verification state, ownership, and the audit trail begin; the zone at the provider does not exist until an API call succeeds, and that call can fail, time out after the write landed, or be retried by an impatient operator. A primary key cannot be null, so teams that reach for the zone id end up with a second `pending_domains` table, and then with two places to look, and then with a verification email sent for a domain that has no row in the table anyone queries.

They are provider-scoped. The handle is opaque and it means nothing outside the account that issued it, so a migration rewrites every foreign key that ever referenced it.

And they churn. Deleting and recreating a zone — the standard support move when a zone is in a state nobody wants to debug — issues a fresh id for the same apex, which silently detaches every historical record operation from the domain it belonged to.

| Primary key choice | Holds up when | Comes apart when |
| --- | --- | --- |
| Application-generated id, `(provider, zone_id)` unique where not null | Domains exist before zones; providers change; evidence joins by name | Nothing, apart from one extra lookup on the write path |
| Provider zone id as the key | Single provider, zones created synchronously, no history kept | Zone is recreated, provider is swapped, or the row must exist first |
| Composite `(provider, zone_id)` | Multi-provider inventory with no pre-zone state | The same apex is migrated between providers and history must survive |
| Apex name as the key | Small, stable inventories | A seller renames or re-registers, and case or punycode normalization slips |

Hosted DNS APIs differ in how much they lean on that handle — Route 53 addresses a hosted zone by an opaque id, Cloudflare by a 32-character hex zone id — and that difference is exactly the coupling you don't want to encode in your own key space.

The read path doesn't want the zone id either. Your hot lookup is an inbound `Host` header resolved to a tenant, which is a lookup by normalized name; your evidence pipeline joins by name too, because that is what the reports carry.

```python
import uuid
import idna

DDL = """
create table domain (
    id         uuid primary key,
    tenant_id  uuid not null,
    name       text not null,   -- normalized apex, <= 253 chars in presentation form
    provider   text,            -- null until a zone exists somewhere
    zone_id    text,            -- provider-scoped handle, an attribute and never an identity
    state      text not null,   -- pending | live | detached
    created_at timestamptz not null default now()
);

create unique index domain_name_uk on domain (name);
create unique index domain_zone_uk on domain (provider, zone_id) where zone_id is not null;
create index domain_tenant_ix on domain (tenant_id);
"""


def normalize_apex(name: str) -> str:
    # DNS comparison is case-insensitive (RFC 4343), and the presentation form caps at 253 chars.
    folded = name.strip().rstrip(".").lower()
    ascii_form = idna.encode(folded, uts46=True).decode("ascii")
    if len(ascii_form) > 253:
        raise ValueError(f"apex too long: {len(ascii_form)} chars")
    return ascii_form


def create_domain(cur, tenant_id: str, apex: str) -> str:
    name = normalize_apex(apex)
    cur.execute(
        "insert into domain (id, tenant_id, name, state) values (%s, %s, %s, 'pending')"
        " on conflict (name) do nothing returning id",
        (str(uuid.uuid4()), tenant_id, name),
    )
    row = cur.fetchone()
    if row:
        return row[0]
    cur.execute("select id from domain where name = %s", (name,))
    return cur.fetchone()[0]
```

## Two failure modes that decide the key

The first one is split-brain zones, and it is boring right up until deliverability evidence starts disagreeing with your dashboard. A seller's apex ends up with two zones at the provider — one created by your onboarding job, one created by hand during a support session — and only one of them is referenced by the delegation. Writes keyed on a stored zone id keep succeeding, with 200 responses and no errors anywhere in your logs, because the API is perfectly happy to accept records into a zone that nothing resolves. The DKIM selector TXT from RFC 6376 lands in the dormant zone, the signature can't be verified by any receiver, and the first honest signal you get is `dkim=fail` in an aggregate report two days later. A uniqueness constraint on the normalized apex in your own database would have refused the second row; a reconciler that lists zones by name and refuses to write when the provider returns more than one would have caught the rest.

The second is partial RRset deletion. Providers expose per-record identifiers, and it is natural to delete the record you created by its id — which leaves the other member of the set in place. One stale SPF TXT beside the new one puts the domain into permerror under RFC 7208, which requires that exactly one SPF record be selected, and the same section caps the record at ten DNS-querying mechanisms, a ceiling you can breach just by adding your platform's `include:` on top of what the seller already had. Receivers then apply the seller's DMARC policy to mail that no longer authenticates.

Record operations should therefore address the set, not the member: the natural unique key is `(domain_id, name, type)`, which is the RRset identity DNS itself uses, and every write should be a replacement of the full set.

```python
import hashlib
import json


def rrset_hash(records: list[dict]) -> str:
    # Fold the owner name only. Case-folding the value would collide two different
    # DKIM keys, and the reconciler would then report "no change" on a key rotation.
    canonical = sorted((r["value"].strip(), int(r["ttl"])) for r in records)
    return hashlib.sha256(json.dumps(canonical).encode()).hexdigest()


def apply_rrset(cur, provider, domain_id: str, name: str, rtype: str, records: list[dict]) -> bool:
    digest = rrset_hash(records)
    cur.execute(
        "select content_hash from rrset where domain_id = %s and name = %s and type = %s for update",
        (domain_id, name, rtype),
    )
    row = cur.fetchone()
    if row and row[0] == digest:
        return False  # no provider call, no journal row, no storage spent

    provider.replace_rrset(domain_id, name, rtype, records)  # whole set, never one member
    cur.execute(
        "insert into rrset (domain_id, name, type, content_hash, records, observed_at)"
        " values (%s, %s, %s, %s, %s, now())"
        " on conflict (domain_id, name, type) do update set"
        " content_hash = excluded.content_hash, records = excluded.records, observed_at = now()",
        (domain_id, name, rtype, digest, json.dumps(records)),
    )
    cur.execute(
        "insert into rrset_change (domain_id, name, type, content_hash, changed_at)"
        " values (%s, %s, %s, %s, now())",
        (domain_id, name, rtype, digest),
    )
    return True
```

## Evidence arrives for domains that have no zone yet

This is the part that settles the argument for a marketplace whose decision axis is deliverability evidence rather than provisioning speed. DMARC aggregate reports identify the domain by name in `policy_published`, and SMTP TLS reports under RFC 8460 do the same; neither format has any idea what a zone id is. Your evidence writer therefore joins on the normalized apex, and it has to be able to land a row for a domain that is still in verification — reports show up for domains whose owners pointed MX at you and then wandered off, and those are often the most diagnostic ones you'll receive.

Keying the fact table on the name and backfilling `domain_id` later costs one nullable column. Keying it on the zone id costs you the evidence.

```python
def record_aggregate_report(cur, report: dict) -> None:
    apex = normalize_apex(report["policy_published"]["domain"])
    cur.execute("select id from domain where name = %s", (apex,))
    row = cur.fetchone()
    domain_id = row[0] if row else None  # evidence can precede the zone, and often does

    for rec in report["records"]:
        cur.execute(
            "insert into dmarc_day"
            " (domain_id, apex, day, source_ip, spf_result, dkim_result, message_count)"
            " values (%s, %s, %s, %s, %s, %s, %s)"
            " on conflict (apex, day, source_ip, spf_result, dkim_result) do update set"
            " message_count = dmarc_day.message_count + excluded.message_count",
            (
                domain_id,
                apex,
                report["date_range_begin"].date(),
                rec["source_ip"],
                rec["auth_results"]["spf"],
                rec["auth_results"]["dkim"],
                rec["count"],
            ),
        )
```

Normalization is where this join quietly rots, by the way. Mixed case from one reporter, a trailing dot from another, a U-label in one feed and its A-label in the next, and you now have three rows for one seller and a chart that undercounts every one of them. Normalize on the way in, store the ASCII form, and make the unique index do the arguing.

## What I stop keeping, and what that costs during a dispute

Raw aggregate XML goes to object storage with a 30-day lifecycle, and the rolled-up daily facts stay for 13 months so that a year-over-year comparison is still possible. Per-poll observations are never written at all — only transitions. Provider request and response bodies are kept for writes and discarded for reads, since reads are the overwhelming majority of calls and carry nothing you'd ever cite. The change journal itself stays indefinitely, because at a few hundred rows a day it is the one thing in this system that is genuinely cheap to keep.

The catch is that you are buying storage back with forensic detail, and the invoice arrives during an argument. Six months on, when a seller insists your platform deleted their MX records, you can prove the RRset hash changed at 04:12 UTC, which actor triggered it, and what the set looked like before and after. You cannot replay the exact bytes the provider received, or its exact response. I've come to think that trade is correct — the intended state plus a hash transition answers nearly every real dispute — but I'm not certain it holds under a regulator who wants original artifacts, and if your compliance regime demands raw evidence for years, keep the XML and pay for it.

One more boundary, because this design is not universal. If you operate authoritative DNS yourself instead of renting it, the zone genuinely is your aggregate root: it has an SOA, a serial, and a lifecycle you control, and a zone-scoped identity is the right model. Same if you are a registrar. This shape is for the other case — the application that borrows zones from someone else's API to make a customer's domain work, where the provider is an implementation detail you should assume you will replace at least once.

## Further reading

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- RFC 7208, Sender Policy Framework (SPF), including the ten-lookup limit: https://datatracker.ietf.org/doc/html/rfc7208
- RFC 6376, DomainKeys Identified Mail (DKIM) Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- RFC 8460, SMTP TLS Reporting: https://datatracker.ietf.org/doc/html/rfc8460
- RFC 1034, Domain Names — Concepts and Facilities: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 4343, Domain Name System (DNS) Case Insensitivity Clarification: https://datatracker.ietf.org/doc/html/rfc4343
- RFC 8555, Automatic Certificate Management Environment (ACME): https://datatracker.ietf.org/doc/html/rfc8555
