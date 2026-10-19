# Osysharp.WorkQueues.Notifications

The people `Osysharp.WorkQueues` says should know, told in their `Osysharp.Notifications` bell: the one an item is offered
to, the cover it is handed to while its holder is away, whoever holds it when its policy says a promise is at risk or
missed, and a queue's leads when nobody picks an item up. Each reads it in their own language, in the second person.

**Use case.** A request is offered to Ines as her account's owner. Her bell says "Offered to you — it is yours if you
take it within 15 minutes." She goes on leave, and her open work is handed to Rahul as cover; his bell says "Handed to
you — Ines is away until Mon 8 Jun 06:00 UTC." The first reply on one of them goes at risk, and its policy says to tell
the holder; his bell says "First reply is at risk." Nobody wrote a line of code for any of it.

```osy
use Osysharp.WorkQueues@1;
use Osysharp.Notifications@1;
// Osysharp.WorkQueues.Notifications is not written: it arrives by itself, because the app uses what it completes (`osy lock` says so)
```

**The leads are whoever you say, once.** Two of the inbox's events are for a queue's leads, `PickupOverdue` and
`PromiseToldToLeads`. You already tell the inbox who leads it, and that is all it takes: the queue's members who satisfy
your policy are told, in the same words — or, when nobody in that queue leads, everybody who leads in any queue.

```osy
policy LeadsInbox => IsLead;            // the one line about your leads, which the inbox needs anyway
```

**Never about an item they may not read.** Every notification is about the item. It is delivered, shown and mailed only
if the recipient may read that item under their own rules. The inbox may name somebody who may not read it, such as an
offer on a restricted item, and that person's bell stays empty.

One unread notification per item and kind of event: a second "at risk" on the same item moves the unread one to the top
instead of adding another.
