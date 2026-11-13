# EVENTS.md

---

## Milestone 1: what runs in the request path, and what runs after it

### The operations table

| # | Operation | As wired today | Your call | Why (in one line) |
|---|---|---|---|---|
| 1 | Validate, check for clashes, save the booking (`book`) | Sync | Sync | The caller needs a yes/no and a booking id right now. |
| 2 | Index the booking for the room-finder (`searchIndex.indexBooking`) | Sync, inside `book` | **Event** (subscriber on `booking.confirmed`) | Only the room-finder page needs it, and a few seconds of lag is fine. |
| 3 | Write the audit record (`audit.record`) | Sync | Sync | The audit record has to match the change exactly. It is local and cheap. |
| 4 | Confirmation and cancellation emails | Event | Event | The organizer can wait a few seconds, and a mail outage should not block a booking. |
| 5 | Render the receipt | Event | Event | The receipt is downloaded later, not during the booking. |
| 6 | Schedule or drop the reminder | Event | Event | The reminder fires 30 minutes before the start, so it has plenty of slack. |
| 7 | Update the dashboard stats | Event | Event | It is a recount of the store, and stale for a moment is harmless. |
| 8 | Refund: write the ledger entry and pay through the provider | Event (`refund.requested`) | Event | The caller needs to know the amount owed. It does not need the money to have moved yet. |

### Two calls in detail

**Search indexing (row 2): move it to the bus.** The page and the kiosk calling
`book()` need three things right away: the request is valid, the room is free,
and the booking is saved with an id. The clash check reads the **store**, not
the search index, so the index adds nothing to whether a booking is correct. It
only feeds the room-finder page. Right now, when the search cluster is down,
`indexBooking` throws, `book()` throws, and the user gets an error. Nothing is
saved. If indexing were a subscriber on `booking.confirmed`, then during an
outage the booking would still succeed and the index write would land in the
dead-letter list. Once search is back, an operator redelivers it. This is safe
to replay because `indexBooking` overwrites the row keyed by booking id. The
cost: the room-finder may not show a new booking for a while. It was already
blank during the outage, so nobody loses anything.

**Refund (row 8): keep it on the bus.** The person cancelling needs two things
right away: the booking released, and how much they will get back. `cancel()`
works both out synchronously and returns them. Actually moving the money
depends on an outside payment provider that can be slow or down. A provider
outage should not make a cancellation fail. The room should still be freed,
and the money can follow later. The price of putting it on the bus: the
handler can be delivered more than once (the bus is at-least-once), so it has
to be idempotent, and somebody has to watch the dead-letter list. Milestones
2 and 3 show both.

### The incident

**The call behind `TICKET.md`.** It is `src/service/bookingService.ts`, in
`BookingService.book()`, at
[line 94](src/service/bookingService.ts#L94):

```ts
this.searchIndex.indexBooking(booking, room, now);
```

When the cluster is down, this throws `SearchIndexUnavailableError`. Nothing
catches it, and it runs **before** `store.save`, so the caller of `book()`
gets that exception with no booking saved and no event published. That is
why all 22 booking attempts failed. Cancellations kept working because
`cancel()` never calls the search index, and earlier confirmation emails
still went out because they come from the bus, not from search.

---

## Milestone 2: what a second delivery does

### Your prediction (before you run anything)

The refund handler has no memory of what it has seen. It appends a ledger
entry and then calls the provider, every time. So I expect each redelivery of
the refund message to:

- **Ledger:** add another $85.00 entry for the same booking (1, then 2, then 3 entries).
- **Provider:** settle another $85.00 (1, then 2, then 3 settlements, so $255 paid back on an $85 booking).
- **Bus log:** still one message, `msg-3`, with `deliveries=3`. No dead letters, because nothing throws.

### Before your fix

**The run.**

```
booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3

=== after the cancellation (one delivery of the refund message) ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3

=== after one redelivery ===
ledger entries: 2
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-2 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 2
gateway settlements: 2
  stl-1 bk-1001 $85.00
  stl-2 bk-1001 $85.00
money returned in total: $170.00
dead letters: 0

redelivering msg-3 again

=== after two redeliveries ===
ledger entries: 3
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-2 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-3 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 3
gateway settlements: 3
  stl-1 bk-1001 $85.00
  stl-2 bk-1001 $85.00
  stl-3 bk-1001 $85.00
money returned in total: $255.00
dead letters: 0

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=3
```

**Your prediction against the run.** They match on every point: one extra
ledger entry and one extra real payment per redelivery, $255 paid out for an
$85 refund, `deliveries=3`, and no dead letters. The bus did exactly what its
contract says. The bug is in the handler.

### Your fix

**The diff.**

```diff
--- a/src/handlers/refundLedger.ts
+++ b/src/handlers/refundLedger.ts
@@ -24,6 +24,10 @@ export class RefundLedger implements Subscriber<RefundRequested> {
   readonly name = 'RefundLedger';
 
   private readonly entries: LedgerEntry[] = [];
+  /** Message ids whose ledger entry has been written. */
+  private readonly recorded = new Set<string>();
+  /** Message ids the payment provider has settled. */
+  private readonly paid = new Set<string>();
   private readonly gateway: PaymentGateway;
 
   constructor(gateway: PaymentGateway) {
@@ -32,15 +36,23 @@ export class RefundLedger implements Subscriber<RefundRequested> {
 
   handle(message: Message<RefundRequested>): void {
     const event = message.payload;
-    this.entries.push({
-      entryId: `led-${this.entries.length + 1}`,
-      bookingId: event.bookingId,
-      organizer: event.organizer,
-      amountCents: event.amountCents,
-      reason: event.reason,
-      recordedAtMinute: message.publishedAtMinute,
-    });
-    this.gateway.refund(event.bookingId, event.amountCents);
+    // Idempotency key: the message id. Each step is remembered on its own, so a
+    // redelivery skips what already happened and finishes what did not.
+    if (!this.recorded.has(message.id)) {
+      this.entries.push({
+        entryId: `led-${this.entries.length + 1}`,
+        bookingId: event.bookingId,
+        organizer: event.organizer,
+        amountCents: event.amountCents,
+        reason: event.reason,
+        recordedAtMinute: message.publishedAtMinute,
+      });
+      this.recorded.add(message.id);
+    }
+    if (!this.paid.has(message.id)) {
+      this.gateway.refund(event.bookingId, event.amountCents);
+      this.paid.add(message.id);
+    }
   }
```

**What your fix uses to decide what to do on a second delivery, and what it
does not protect.** The key is the **message id**, tracked separately for "ledger
written" and "provider paid". A redelivery skips the steps that already
finished and redoes only the ones that did not. One "seen" flag would have
been wrong: after a half-finished delivery it would skip the payment forever
(milestone 3). What it does **not** protect against:

- A second, *different* refund message for the same booking. It gets a new id, so it pays again.
- A restart. The sets live in memory and are lost.
- A crash right after the provider settles but before `paid.add`. The provider pays twice. Fixing that needs the provider itself to take an idempotency key.

### After your fix

**The run.**

```
booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3

=== after the cancellation (one delivery of the refund message) ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3

=== after one redelivery ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3 again

=== after two redeliveries ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=3
```

**What stays the same across its three states.** These numbers are identical
in all three states: ledger entries = 1, ledger total = $85.00, gateway
calls = 1, settlements = 1, money returned = $85.00, dead letters = 0. Only
the bus's `deliveries` count goes up, to 3. The optional fast-check property
in `tests/optionalProperty.test.ts` is turned on and passes (34/34 tests).

---

## Milestone 3: a delivery that stops halfway

### The failure

**The run** (`npm run demo:failure`, the part before the recovery):

```
armed: call 1 to the payment provider will throw
booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3
the cancel call itself returned: booking is cancelled

=== after the failed delivery ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 0
money returned in total: $0.00
dead letters: 1
  msg-3 refund.requested attempt 1 in RefundLedger: payment provider did not answer on call 1 for bk-1001
    failedAtMinute: 480
    payload: {"bookingId":"bk-1001","organizer":"H. Ahmad","organizerEmail":"hahmad@example.edu","amountCents":8500,"reason":"cancelled more than two hours ahead","requestedAtMinute":480}

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=1
```

### Your recovery

**What, if anything, the failed delivery left undone.** The handler got
halfway. The ledger entry was written (the ledger says $85 refunded), but the
provider threw, so **no money moved** ($0 settled). The ledger and the provider
now disagree, and the organizer has not been paid. The cancel itself
succeeded, because `publish` does not throw.

**The option I took: redeliver.** `failureDemo.ts` now calls
`app.bus.redeliver(cancellation.refundMessageId)` at the end of `main()` and
prints the state again:

```
redelivering msg-3

=== after redelivering the dead-lettered message ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 2
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 1
  msg-3 refund.requested attempt 1 in RefundLedger: payment provider did not answer on call 1 for bk-1001
    failedAtMinute: 480
    payload: {"bookingId":"bk-1001","organizer":"H. Ahmad","organizerEmail":"hahmad@example.edu","amountCents":8500,"reason":"cancelled more than two hours ahead","requestedAtMinute":480}

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=2
```

Bus log line for the refund message: `msg-3 refund.requested deliveries=2`.
Delivery 1 failed and delivery 2 finished the job: still **one** ledger entry,
and now **one** $85 settlement. The dead letter is still listed because the
list is a log of failures, not a queue. Redelivery does not remove it.

This only works because of the milestone 2 fix. The original handler would
have written a second ledger entry ($170 in the ledger, $85 paid). A naive
"skip if seen" fix would have skipped the payment and never paid at all.

`npm run typecheck` and `npm test` both stay green.

### The other option

**What the option I did not take would have cost.** Leaving the message for
triage puts the work on a human, so the organizer waits until someone reads
the dead-letter list (hours or days, not seconds). It is only safe if that
person first checks the provider for an existing settlement for `bk-1001` and
pays without adding a second ledger entry, since the ledger already shows the
refund.
