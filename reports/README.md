# Osysharp.WorkQueues.Reports

What your team's inbox did, as reports: how much came in, how much was done, how soon the first reply came and how long
finishing took (the median and the slowest tenth), what stands open now, and how many promises were kept. By agent,
queue, channel, organization, tier and priority, over any item kind your app works through `Osysharp.WorkQueues`.

It is the bridge between two kits, so neither depends on the other: an app that only works a queue does not get
reports, and `Osysharp.Reports` knows nothing about inboxes. Add both and say, in your own words, what you report on:

```osy
use Osysharp.WorkQueues@1;
use Osysharp.Reports@1;
// Osysharp.WorkQueues.Reports is not written: it arrives by itself, because the app uses what it completes (`osy lock` says so)
use Osysharp.Explore;

public InboxReportWords DeskWords() {
  return new InboxReportWords { Title = Translate("Tickets"), Noun = Translate("ticket"), Nouns = Translate("tickets"),
                                Done = Translate("Solved"), Finish = Translate("solve", key: "report-finish-verb"),
                                Organization = Translate("Company") };
}

public class TicketReports : IReportSource {
  public ReportSource Describe() { return InboxWorkSource("tickets", Explore.Root<Ticket>(), DeskWords()); }
}

public class PromiseReports : IReportSource {
  public ReportSource Describe() { return InboxPromiseSource("promises", DeskWords()); }
}

// The overview: tickets and the promises kept on them, side by side, by agent, queue, company…
ReportOverview("tickets", ["arrived", "done", "firstReply", "toDone", "promises:keptShare"], now: ["open"]);
```

Both sources take the same words, so one overview can put a figure from each in one breakdown: it joins them on the
grouping with the same name.

## What it counts, exactly

| | |
|---|---|
| **The work, by when it arrived** | `arrived`, `done` (done or closed, not cancelled), `doneShare`. "Of what arrived in March, how much is done": a figure never changes what it divides by under the reader |
| **First reply and time to done** | `firstReply`, `firstReplySlowest`, `toDone`, `toDoneSlowest`: the median and the 90th percentile of the calendar time from arrival, the time the requester actually waited |
| **Open now** | `open`: not done, closed or cancelled, whenever it arrived. Show it as an overview's `now`, where the period does not apply |
| **Promises, by when they started** | `ended`, `kept`, `keptShare`, `missed`: a promise counts once it has ended, met in time or late. One still running is neither kept nor missed yet, and a cancelled one measured nothing |
| **In business hours** | `firstReplyHours`, `toDoneHours`: the same medians as the promise clocks counted them, in the item's own business hours |
| **Groupings** | agent, queue, channel, organization, tier and priority, under your words, on both sources; and the promise itself |
| **As the reader may read** | an item in a restricted queue counts for nobody outside it, and neither do its promises |

| The helpdesk template's overview | …in Arabic |
|---|---|
| ![a support desk's overview: tickets, solved, first reply, time to solve, promises kept, satisfaction](docs/inbox-reports-overview.png) | ![the same overview, right to left in Arabic](docs/inbox-reports-overview-arabic.png) |

## Its tests

The kit's test app is a building's maintenance desk: tenants report a leak or a dead light, technicians take the job,
and every promise is counted in office hours. A week of jobs is played through the inbox's real runs (round robin,
first replies, fixes, a promise answered late, one still owed) and every figure is checked against an answer worked out
by hand. A job in a restricted queue is shown to count for nobody outside it, its promises included.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.WorkQueues;
using Osysharp.WorkQueues.Reports;
using Osysharp.Reports;
using Osysharp.Explore;
using InboxReportsKitTests;

[Test(Desk)]
[runas(Lena)]
void a_job_in_a_restricted_queue_counts_only_for_those_who_may_see_it() {
  TestClock.Set(new DateTime(2026, 6, 3, 9, 0, 0));
  Report("Master key missing", "Locks");
  TestClock.Set(new DateTime(2026, 6, 3, 10, 0, 0));
  var promises = new ExploreQuery { Root = Explore.Root<SlaClock>(), Measures = [new ExploreMeasure { Name = "n" }],
                                    GroupBy = [new ExploreGroup { Field = "Item.Queue" }] };
  var jobs = new ExploreQuery { Root = Explore.Root<Job>(), Measures = [new ExploreMeasure { Name = "n" }] };
  // Lena works Locks: the job, and its two running promises (first reply, fix).
  Assert.Equal(1m, Explore.Run(jobs).Rows[0].Cells[0].Number);
  var mine = Explore.Run(promises);
  Assert.Equal("Locks", mine.Rows[0].Cells[0].Text);
  Assert.Equal(2m, mine.Rows[0].Cells[1].Number);
  // Rahul does not: none of it, anywhere.
  runas (Rahul) {
    Assert.Equal(0m, Explore.Run(jobs).Rows[0].Cells[0].Number);
    Assert.Equal(0, Explore.Run(promises).Rows.Count);
  }
}
```

From [`tests/security.test.osy`](tests/security.test.osy).

```osy
using Osysharp.WorkQueues;
using Osysharp.WorkQueues.Reports;
using Osysharp.Reports;
using Osysharp.Explore;
using InboxReportsKitTests;

[Test(Desk)]
[runas(Lena)]
void the_week_reads_as_worked_out_by_hand() {
  TestClock.Set(new DateTime(2026, 6, 1, 9, 0, 0));
  var j1 = Report("Leak under the sink, room 204", "Plumbing");                    // Ines's turn
  TestClock.Set(new DateTime(2026, 6, 1, 10, 0, 0)); Settle([j1]);
  Responded(j1); Settle([j1]);
  TestClock.Set(new DateTime(2026, 6, 1, 13, 0, 0)); Settle([j1]);
  WorkDone(j1); Settle([j1]);

  TestClock.Set(new DateTime(2026, 6, 1, 15, 0, 0));
  var j2 = Report("Toilet runs all night, 3rd floor", "Plumbing");                 // Rahul's
  TestClock.Set(new DateTime(2026, 6, 2, 9, 0, 0)); Settle([j2]);
  var j3 = Report("Burst pipe in the stairwell", "Plumbing");                      // Ines's again
  TestClock.Set(new DateTime(2026, 6, 2, 10, 0, 0)); Settle([j2, j3]);
  Responded(j2); Settle([j2]);
  TestClock.Set(new DateTime(2026, 6, 2, 15, 0, 0)); Settle([j2, j3]);
  WorkDone(j2); Settle([j2]);
  TestClock.Set(new DateTime(2026, 6, 2, 18, 0, 0)); Settle([j2, j3]);
  Responded(j3); Settle([j3]);

  TestClock.Set(new DateTime(2026, 6, 3, 9, 0, 0));
  var j4 = Report("Socket sparks in the kitchen", "Electrical");
  TestClock.Set(new DateTime(2026, 6, 5, 12, 0, 0)); Settle([j1, j2, j3, j4]);

  // The work, counted by when it arrived.
  Assert.Equal("4", Figure("jobs", "arrived"));
  Assert.Equal("2", Figure("jobs", "done"));
  Assert.Equal("50%", Figure("jobs", "doneShare"));

  // First reply and time to fix: the calendar time the tenant waited. First replies 1h, 19h, 9h — the median is 9h and
  // the slowest tenth 9h + 0.8 × 10h; fixes 4h and 24h — the median of two is their middle.
  Assert.Equal("9 h", Figure("jobs", "firstReply"));
  Assert.Equal("17 h", Figure("jobs", "firstReplySlowest"));
  Assert.Equal("14 h", Figure("jobs", "toDone"));

  // Open: what stands open now, whenever it arrived.
  var s = ReportSourceOrNull("jobs")!;
  var open = RunSourceNow("jobs", new ExploreQuery { Root = s.Root, Measures = [s.Measures.Single(m => m.Measure.Name == "open").Measure] });
  Assert.Equal(2m, open.Rows[0].Cells[0].Number);

  // A promise is kept or missed once it has ended, and measured in business hours: first replies 1h, 3h (j2's 19
  // calendar hours held 3 office hours), 8h; fixes 4h and 8h.
  Assert.Equal("5", Figure("promises", "ended"));
  Assert.Equal("4", Figure("promises", "kept"));
  Assert.Equal("80%", Figure("promises", "keptShare"));
  Assert.Equal("4", Figure("promises", "missed"));
  Assert.Equal("3 h", Figure("promises", "firstReplyHours"));
  Assert.Equal("6 h", Figure("promises", "toDoneHours"));

  // The two sources group by the same words — which is what lets one overview join them.
  Assert.Equal("– | 1 | 0\nInes | 2 | 1\nRahul | 1 | 1", By("jobs", "Technician", ["arrived", "done"]));
  Assert.Equal("– | 0 | 0\nInes | 3 | 2\nRahul | 2 | 2", By("promises", "Technician", ["ended", "kept"]));
  Assert.Equal("Electrical | 1\nPlumbing | 3", By("jobs", "Trade", ["arrived"]));
  // By the organisation the work is for, and by the tier it is on — read through the Accounts add-on's CustomerAccount:
  // `WorkItem.Organization` is a customer account, so its `Tier` is one relation further.
  Assert.Equal("Larkspur | 4", By("jobs", "Tenant", ["arrived"]));
  Assert.Equal("Standard | 4", By("jobs", "Tier", ["arrived"]));
  Assert.Equal("Standard | 5 | 4", By("promises", "Tier", ["ended", "kept"]));
}
```

From [`tests/figures.test.osy`](tests/figures.test.osy).

## Learn more

- [The Osysharp.WorkQueues.Reports reference](https://osysharp.com/reference/ui/reports-kit/)
- [Osysharp.WorkQueues.Reports in the package catalogue](https://osysharp.com/packages/#osysharp-workqueues-reports)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
