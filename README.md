# Osysharp.WorkQueues

Work queues, who holds what, and the service levels promised on it — over **any** item: a ticket, an approval, a
candidacy, a lead, a return, an alert.

An app's item joins the inbox by deriving from **`WorkItem`**. The app keeps what the item *is* — its own states, its
fields, its guards. It tells the kit four facts as they happen, and the kit keeps the time:

| the app says | when | what the kit does |
|---|---|---|
| `Responded(item)` | the team gave the requester a response | meets the first response, or the response owed since they last wrote |
| `RequesterWrote(item)` | the requester wrote | a response is owed again — a new next-response clock |
| `WaitOn(item, WaitFor.Requester / Others / Pause)` · `ResumeWork(item)` | the app's own state means waiting | waiting on the requester (or held) pauses every clock; waiting on others keeps resolution running unless the policy pauses it |
| `WorkDone(item)` · `ReopenWork` · `CloseWork` · `CancelWork(item, why)` | solved / reopened / closed / cancelled | meets resolution; a reopen resumes it from where it stopped |

Each takes `at:` to replay an imported item's history as it really happened. `Responded` and `Prioritise` are the
team's (`WorksInbox`) — a requester cannot stop the team's clock or raise their own priority.

## The promise

- **`SlaPolicy`** — conditions (priority, the organisation's tier, queue, channel, segment, kind, VIP) and the hours its
  clocks run in. The **most specific** active policy whose every stated condition matches wins; ties by `Position`.
  An item no policy matches has **no promise**, and `Clocks(item)` is empty rather than a zero.
- **`SlaTarget`** — one per metric: `FirstResponse`, `NextResponse`, `Resolution`. The app's own words for them come
  from `InboxWords` ("feedback due", "decide by").
- **`SlaClock`** — the record of each promise on each item (every next response is its own cycle): when it started, the
  business time it used, when it went at risk, was met, was breached. A breach is **recorded, never fatal** — the
  requester still has their problem.
- `Prioritise(item, priority)` chooses the policy again and **retargets** every running clock, keeping the time used.

## Queues, and who holds an item

- **`Queue`** / **`QueueMember`** — where work waits, and who works it. An item lands in the queue the app names, else
  `InboxSetup.DefaultQueue`. A `Restricted` queue's items are invisible — in lists and counts — to staff outside it.
- **`TakeWork(item, expected)`**, `ReleaseWork`, `HandWorkTo(item, person, expected)`, `MoveWork(item, queue)`,
  `TakeMany`, `MoveMany` — only somebody who **may hold** an item (an active member of its queue) is ever given it.
  `expected` is the holder the page showed: a take that lost the race is refused with "Rahul took this just now."
  A lead moves anybody's work; an agent only their own unless `InboxSetup.AgentsHandOn`.
- **`Handover`** — every change of holder or queue, with its reason and who did it. `TrailOf(item)` is "why is this
  here".
- **`Addressee`** — an item made for one named person (a question put to them, an approval that is theirs) is given
  to them as it arrives, when they may hold it, instead of being routed.

## Who is given it

- **`RoutingRule`** — an admin's ordered rules: conditions (channel, words in the title, organisation, tier, VIP,
  language, segment, kind) and a closed set of actions (queue, priority, a needed skill, a person). The first match
  stops unless it says `ContinueAfter`. The app's own follow-ups go in `InboxRouting.Routed`.
- **`Queue.Distribution`** — `Manual` (the pool; `PickupWithin` tells the leads), `RoundRobin` (each free member in
  turn) or `LeastLoaded` (least work for their capacity). Free = may hold it, on duty (`Availability`: available, inside
  their own working hours), below capacity (the least of their own, their membership's and the queue's). Load counts
  only work that needs them — not waiting on the requester, not done.
- **Skills** — `RoutingSkill`, `HeldSkill` (with a level), `NeededSkill`. A required skill limits who may hold an item;
  a preferred one ranks who is given it. An item's language is a preferred skill: for the queue's `LanguageWait` only
  speakers are given it, then anybody free.
- **Refill** — nobody free, it waits in the pool, and is pulled the moment somebody is: their load drops, or the
  language wait ends. Nothing polls.
- **What happens is an event** — `ItemArrived`, `ItemQueued`, `ItemRouted` (a routing rule placed it), `HolderChanged`, `ItemMoved`, `ItemEscalated`,
  `ItemOffered`, `OfferFellThrough`, `OfferWithdrawn`, `RoutingSkipped`, `ItemPooled`, `PickupOverdue`,
  `PriorityChanged`, `DueMoved`, `ItemCancelled`, `ItemReopened`, `PromiseAtRisk`, `PromiseBreached`, `PromiseMet`,
  `PromiseToldToHolder`, `PromiseToldToLeads`, `ItemLinked`, `ItemUnlinked` — each about the item, each saying itself in
  the reader's language. Automations runs rules on them, Webhooks sends them, Activity lines them up, and an app hears
  any of them with `[On]`, with nothing registered. Who should know is on the event (`ItemOffered.To`,
  `HolderChanged.Told`, `PromiseToldToHolder.Holder`); `Osysharp.WorkQueues.Notifications` tells them in their bell, and the
  app names its leads for `PickupOverdue` and `PromiseToldToLeads`.

## The account's own people first, and time away

- **Account routing** — an organisation's item is offered to its account team before the queue hands it out: owner →
  backup (or the owner's cover) → pod → the segment queue (`ForTier` / `ForSegment`) → the queue. Each `RoutingOffer`
  waits the queue's `OfferFor` in the queue's hours (`OfferFlow`); anybody who cannot take it now is skipped at once,
  with the reason on the trail. Taking or responding accepts; `DeclineOffer` falls through. `MyAccounts()` and
  `Rows(BuiltInView.MyAccounts)` are the viewer's accounts.
- **Away** — `GoAway(until, cover)` / `ComeBack()`, a lead's `MarkAway` / `MarkBack`, or a planned `Absence`
  (`AbsenceFlow`): held work goes by the queue's `OnAway` rule — to the cover when the cover may hold it, else back to
  the queue — and offers fall through. What cover took stays with the cover. `ActsFor(u, p)` is delegation.
- **Follow-the-sun** — halfway to the first-response promise, an item whose holder is off their hours goes to the least
  loaded member on theirs.

## Escalation

- **Levels** — `Queue.Level` and `Queue.EscalatesTo`. `EscalateWork(item, why)` moves the item up: its promise is read
  again and retargeted, it is routed again at the new level, and its holder keeps it only where they may hold it.
  Each is an `Escalation` row and a `Handover` (`Escalated`).
- **`SlaEscalation`** — what a policy's clocks do when at risk or breached: `TellHolder`, `TellLeads`, `RaisePriority`,
  `Escalate`, `MoveTo`. Once per clock cycle. Every promise goes at risk — the resolution too.

## Views and counts

- `Rows(BuiltInView.Mine | Offered | MyAccounts | Unassigned | Open | AtRisk | Waiting | Done)`, `RowsIn(queue)`,
  `RowsOf(savedView)` — a fixed handful of statements however long the list.
- **`Counts()`** — one grouped read, summed into every view's and every queue's count.
- **`InboxView`** — a saved view, personal or shared (admins'), filtering by queue, holder, priority floor, tier,
  organisation, at-risk, language, segment, kind and the app's own filter (`InboxViews.Narrow`).
- Every list and count is the **viewer's own read**: a view grants nothing, and a count never includes what the viewer
  may not read.

## Links between items

- **`InboxSetup.Links`** — the kinds of link the app's items may have, a closed set it declares (`LinkKind`: its key,
  its words from each side, whether it reads the same both ways, whether an item links that way to one other at most,
  whether people choose it or only the app's code makes it).
- **`Link`/`Unlink`** (and `LinkWork`/`UnlinkWork` for a page) — refused with a sentence for an undeclared kind, a self
  link, a link that already stands, a second one-target link, a chain. Both items' history is told. A link moves
  nothing; what a kind means (a problem's incidents solved with it) is the app's, through `LinkedFrom`/`LinkedTo`.
- **`ItemLinks(item, hrefOf)`** — the panel: each link from the item's side, live; "Link…" finds the other item by its
  reference or title words. The team's alone: a requester never reads a link.

## Screens

On the `Osysharp.Ui` floor (Switchboard): `InboxShell(chrome)` puts the views, queues and saved views — with live counts,
lamps on what is yours, and the "go to" keys (G then M/O/U/A/C) — into the app's own sidebar; `WorkList(view, hrefOf,
open)` is the dense live list worked from the keyboard (J/K/X/Enter/T, and a bulk bar); `WorkItemHead(item)` is an item's
head — who else is on it, `AssignControl` (hand it only to somebody who may hold it, with their load), its queue,
priority and `ItemClocks`; `WhyHere(item)` is its trail in sentences; `MyStatus()`, `TeamLoad()`, `MyAccountsPanel()`;
and the settings pages `QueueSettings`, `SlaPolicySettings` (with "which policy applies?"), `BusinessHoursSettings`,
`RoutingRuleSettings`, `SkillSettings`, `SharedViewSettings`. They reflect the app's own `WorksInbox`/`LeadsInbox`/
`AdministersInbox` — nothing to thread through. `osy docs ui-inbox-kit`; the test app's `model/pages.osy` places them all.

### What another kit adds

Two places on these screens are open to other kits, and to the app:

- **An item's detail** — `WorkItemDetail`. The kit that owns a kind of item draws it under the item's head, on every
  screen that shows `WorkItemView(item)`:
  `component ReturnDetail(Return item) : WorkItemDetail<Return> { render { … } }`. An item of a kind nobody draws
  shows the head alone.
- **A change to many items** — `WorkBulkAction`. A button on the list's bulk bar, told the kind the picked items share
  (`WorkItem.Kind`, or "" when they differ) and their ids:
  `component ApproveAll(string kind, List<Guid> ids, Action<string> done) : WorkBulkAction { … }`. Its action calls a
  function of its own kit and then `done("2 of 3 approved.")` — the list says the sentence and clears the pick. The
  function runs as the person pressing: an item they may not change stays as it was.

So an approval, a return or a proposal that arrives from another kit is an ordinary item in the same inbox, shown and
worked the way its own kit decides. The test app's `kits/proposals` is a complete example in sixty lines.

## Use it

```osy
// app.osy
use Osysharp.Identity@0;
use Osysharp.Organizations@0;
use Osysharp.Organizations.ServicePlans@0;   // customers' tiers, account teams, VIPs — read, never written
use Osysharp.Presence@0;   // the "moved first" sentence; seats on an item
use Osysharp.Observability;
use Osysharp.WorkQueues@1;
use Osysharp.Workflow;
use Osysharp.ServiceHours;
```

```osy
using Osysharp.WorkQueues;

// The kit's three questions (and its neighbours': Organizations', Collaboration's).
policy WorksInbox            => IsAgent;
policy LeadsInbox            => IsLead;
policy AdministersInbox      => IsAdmin;
policy ManagesOrganizations  => IsAdmin;
policy CollaboratesOnRecords => WorksInbox;

entity Ticket : WorkItem {
  [MaxLength(4000)] string Body = "";
  security { allow read, create, update when WorksInbox; allow read, create when IsAuthenticated where Requester == user; }
}

public void Reply(Ticket t, string text) { /* post it */ Responded(t); }
```

An item with a lifecycle of its own keeps it: its workflow calls the four verbs from its states (`enter { WaitOn(this.Item,
WaitFor.Requester); }`) and declares no clock of its own — two runs per item, the app's states and the kit's time.

## The floor

`WorkItem`'s denies hold on every subtype whatever the app grants: the holder, queue, priority and promise are written
only by the kit's run; the requester's side never reads the policy, the budgets, the at-risk fraction, the assignment
history or the clock records; a restricted queue is its members' alone; nobody but you or a lead sets your availability; cover goes only to who may hold the work.
Ten security rules, each with its own test in
`tests/security.test.osy`; `osy docs security-inbox-kit` explains them.

## Its tests

The kit's test app carries two kinds of work, `Request` (a requester who waits on us) and `Spend` (an approval with its
own lifecycle), so the kit is proven general on its own, not only by the helpdesk.
