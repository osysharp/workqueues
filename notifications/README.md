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

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.WorkQueues;
using Osysharp.Notifications;
using Osysharp.WorkQueues.Notifications;
using InboxNotificationsKitTests;

[Test(Crew)]
void the_one_an_item_is_offered_to_reads_it_in_their_bell_in_the_second_person() {
  var r = Reported("Streetlight out on Elm Road");
  runas (Ana) {
    Raise(new ItemOffered { Item = r, To = Named("Mia"), Step = new InboxStep { Reason = HandoverReason.AccountOwner, About = "Mia", Minutes = 15 } });
    UnitOfWork.Commit();
  }
  runas (Mia) { Assert.Contains("Offered to you — it is yours if you take it within 15 minutes.", MyBell()); }
  runas (Leo) { Assert.Equal(0, MyBell().Count); }
}
```

From [`tests/told.test.osy`](tests/told.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.WorkQueues;
using Osysharp.Notifications;
using Osysharp.WorkQueues.Notifications;
using InboxNotificationsKitTests;

[Test(Crew)]
void work_handed_to_a_cover_is_in_the_covers_bell_and_a_plain_change_of_holder_tells_nobody() {
  var r = Reported("Blocked drain on Mill Lane");
  runas (Ana) {
    Raise(new HolderChanged { Item = r, Told = Named("Leo"),
                              Step = new InboxStep { Reason = HandoverReason.Cover, From = "Mia", To = "Leo",
                                                     Until = new DateTime(2026, 6, 8, 6, 0, 0) } });
    Raise(new HolderChanged { Item = r, Step = new InboxStep { Reason = HandoverReason.Claimed, To = "Mia" } });
    UnitOfWork.Commit();
  }
  runas (Leo) { Assert.Equal(["Handed to you — Mia is away until Mon 8 Jun 06:00 UTC."], MyBell()); }
  runas (Mia) { Assert.Equal(0, MyBell().Count); }
}
```

From [`tests/told.test.osy`](tests/told.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.WorkQueues;
using Osysharp.Notifications;
using Osysharp.WorkQueues.Notifications;
using InboxNotificationsKitTests;

[Test(Crew)]
void whoever_holds_it_is_told_of_a_promise_when_its_policy_says_so() {
  var r = Reported("Pothole outside the school");
  runas (Ana) {
    Raise(new PromiseToldToHolder { Item = r, Holder = Named("Mia"), About = "Mia", Metric = SlaMetric.FirstResponse,
                                    Moment = SlaMoment.AtRisk });
    UnitOfWork.Commit();
  }
  runas (Mia) { Assert.Equal(["First response is at risk."], MyBell()); }
}
```

From [`tests/told.test.osy`](tests/told.test.osy).

## Learn more

- [The Osysharp.WorkQueues.Notifications reference](https://osysharp.com/reference/workflow/inbox-notifications-kit/)
- [Osysharp.WorkQueues.Notifications in the package catalogue](https://osysharp.com/packages/#osysharp-workqueues-notifications)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
