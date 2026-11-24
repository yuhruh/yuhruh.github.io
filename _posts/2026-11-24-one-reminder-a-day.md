---
layout: post
permalink: /notes/one-reminder-a-day/
title: "One reminder a day, even when the job runs twice"
description: >-
  How Household Tracker uses unique indexes as locks so each family member gets exactly one
  reminder digest per day, and why it only counts a push as delivered when the phone says so.
tags: [Rails, PostgreSQL, Web Push, Reliability]
---

Household Tracker reminds my family about bills before they're due. Every member gets one digest
a day, at the hour they chose and in their own time zone, through up to three channels: the
in-app inbox, email and Web Push.

"One a day" sounds simple. It's easy to get wrong in both directions:

- **Twice.** The hourly job can overlap with itself: a retry, a slow run that's still going when
  the next one starts, or two workers after a deploy. If both decide "Alex hasn't been reminded
  today", Alex gets two emails.
- **Never.** The process can die after deciding to send and before sending. If the system has
  already written down "reminded", nobody tries again.
- **Lying.** A push service answering `201 Created` doesn't mean the phone showed anything. If
  that counts as "sent", the status page is wrong exactly when it matters.

Here is how the app handles each one.

## 1. Let the database decide who wins

The obvious version is check-then-insert (a sketch, not the app's code):

```ruby
unless ReminderDelivery.exists?(user:, channel:, run_on: today)
  ReminderDelivery.create!(...)   # two overlapping runs both get here
end
```

Two runs that start at the same moment both see "nothing yet", and both send. A Ruby-level mutex
doesn't help across processes or machines.

Instead, the rule "one per day" lives in PostgreSQL as unique indexes, at two levels:

```ruby
# One claim per bill, member, reminder stage, channel and day
t.index ["record_id", "user_id", "kind", "channel", "scheduled_on"],
        name: "index_reminder_logs_claim", unique: true

# One digest per household, member, channel and local day (plus at most one same-day addition)
t.index ["household_id", "user_id", "channel", "run_on", "addition"],
        name: "index_reminder_deliveries_one_per_day", unique: true
```

Each run tries to *claim* every item it wants to remind about by inserting a row. The insert
either succeeds, and this run owns that reminder, or raises `RecordNotUnique`, and another run
already owns it:

```ruby
def claim(item, user, channel)
  ReminderLog.transaction(requires_new: true) do
    ReminderLog.create!(record: item.record, user:, kind: item.kind, channel:,
                        scheduled_on: item.scheduled_on, snoozed: item.snoozed)
  end
rescue ActiveRecord::RecordNotUnique
  nil
end
```

`requires_new: true` matters. In PostgreSQL, a failed statement aborts the whole transaction it's
in. The nested transaction becomes a savepoint, so losing one claim rolls back only that insert,
and the run carries on with the items it did win.

The claims and the digest row are created in one transaction. If a second run gets as far as
creating the same day's digest, the second unique index stops it, and the whole attempt rolls back:

```ruby
rescue ActiveRecord::RecordNotUnique
  nil # a concurrent run created today's delivery; its claims won
```

The index is the lock. There's no lock table, no advisory lock, no Redis. The database already has
to keep the data consistent, so I let it enforce the rule as well.

## 2. Recover from a run that died halfway

Claiming and sending are separate steps. The run writes the claims and a `pending` digest in one
transaction, then queues a job to send it. If the process dies after that commit, the digest
exists but nothing is sending it.

Each run starts by picking those up:

```ruby
# Claimed but never dispatched (the run died in between): dispatch on the next run
def recover_stuck_deliveries
  ReminderDelivery.pending.where(channel: %w[email web_push], dispatched_at: nil)
    .where(created_at: ...(@now - STUCK_AFTER))
    .find_each { |delivery| DeliverReminderJob.perform_later(delivery.id) }
end
```

The send job is safe to run again: it returns straight away unless the digest is still `pending`.

To be honest about the limits: the *decision* to remind happens exactly once, but the last hop is
at-least-once. If the process dies after the mail server accepted an email and before the row is
marked `sent`, a retry can send that email again. The design keeps that window as small as it can;
it can't close it, because no system can make an external send and a database write atomic.

## 3. "Accepted" is not "delivered"

Web Push has two legs: my server hands the message to the browser's push service, and the push
service delivers it to the phone whenever the phone wakes up. The `201` I get back only covers the
first leg.

So every push carries a single-use receipt URL. The server stores only a SHA-256 digest of the
token, behind its own unique index. The service worker posts the receipt after it has shown the
notification:

```js
event.waitUntil((async () => {
  await self.registration.showNotification(title, options)
  if (data.receipt_url) {
    try {
      await fetch(data.receipt_url, { method: "POST", keepalive: true, credentials: "omit" })
    } catch (_) {
      // Receipt is best effort; an unconfirmed delivery is reported after 24 h.
    }
  }
})())
```

The receipt endpoint is deliberately unauthenticated and CSRF-exempt, because the phone's session
may have expired. The single-use token is the only credential, and the endpoint is rate limited.
A digest moves `pending → awaiting_receipt → sent` only when a device confirms it. An hourly job
marks anything still unconfirmed after 24 hours as failed, and I see it in the next daily summary.
For "bill paid" and supply alerts, an unconfirmed push falls back to email.

I measured this before relying on it. On a Pixel 9 with the app closed, the push service accepted
every message within a second. With the screen off, one receipt arrived **25.6 minutes** later.
Overnight with battery saver on, one arrived after **112 minutes**. If I had treated `201` as
"delivered", those reminders would have shown as delivered while they were still waiting.

## 4. Pin it down with tests

The behavior is covered at the level a user would describe it:

```ruby
test "nothing before the member's reminder hour; at 08:00 one digest per channel; later runs add nothing" do
  bill
  run_at(taipei(2026, 10, 20, 7, 30))
  assert_empty deliveries
  assert_enqueued_jobs 2, only: DeliverReminderJob do
    run_at(taipei(2026, 10, 20, 8, 5))
  end
  assert_equal %w[in_app email web_push], deliveries.pluck(:channel)
  # … (status and wording checks trimmed)
  run_at(taipei(2026, 10, 20, 9, 5))
  assert_equal 3, deliveries.count
end
```

Alongside it, another test checks that a digest stuck in `pending` for two hours is dispatched
again.

## What I'd take to any project

- **Put the invariant where every writer has to go through it.** A unique index can't be skipped by
  a forgotten code path or a second worker.
- **Name the unit of idempotency.** Here it's *member × household × channel × local day*. Once that's
  written down, the index almost writes itself.
- **Keep "decided", "sent" and "received" as separate states.** Each one fails differently, and
  merging them hides the failures.
- **Measure the edge you don't control** before you design around it.

The code is in the [Household Tracker repository](https://github.com/yuhruh/HouseHold_Notification).
