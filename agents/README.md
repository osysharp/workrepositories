# Osysharp.WorkRepositories.Agents

What an agent finds lands in the assignment's repository as items — not as lines of a trace.

```osy
public class AnalyseSample : Skill<Sample, Verdict> {
  public override List<ActionInfo> Tools => [
    Actions.Of(Readings),                                       // the app's own
    Actions.Of(RecordEvidence), Actions.Of(RecordFinding),      // write what it found
    Actions.Of(ListItems), Actions.Of(ReadItem),                // read what is already there
  ];
}
```

## Use case

An agent looks something up, concludes something, and hands back a result. A day later somebody asks what it based
that on. If the answer is in a transcript, it is gone for practical purposes: nobody can accept it, reject it, set
it against another finding, or keep it for exactly as long as the law allows.

- **It writes items with tools.** Each is added as the agent's own account, on behalf of whoever asked, with where
  it came from, the tool that wrote it, and the piece of work it was written in.
- **It reads by reference.** `ListItems` answers ids, kinds, titles and standings; `ReadItem` answers one. Nothing is
  copied into its instructions, and a tool handed an item whoever asked may not read is handed nothing.
- **What it hands back is kept too.** When a piece of agent work attached to an assignment finishes, its result is
  filed as a finding with the agent, the skill and its version, and the receipt it came from.
- **Nothing it says is taken as right.** Every item an agent adds is unverified until somebody in charge accepts it.
- **A helper gets exactly the items its task needs.** `HandItemsTo(agent, items)` opens them to another agent's
  account, on the assignment or not; `TakeItemsFrom` closes them.

## What it does

| | |
|---|---|
| `RecordEvidence(assignment, title, content, source)` | A piece of evidence. |
| `RecordFinding(assignment, title, content, source)` | What it concludes. |
| `RecordCorrespondence(assignment, title, content, source)` | A message sent or received. |
| `ProposeAction(assignment, title, content)` | A proposal. It decides nothing. |
| `NoteItemsDisagree(item, otherItemId, about)` | Two items that do not agree, kept open until a decision settles it. |
| `ListItems(assignment)` · `ReadItem(item)` | The repository by reference. Read-only. |
| `HandItemsTo(agent, items)` · `TakeItemsFrom(agent, items)` | A scoped reference for a helper. |
| `app.AgentFindings = new AgentFindingsSetup { FileResults = false }` | Switch the filing of results off. |

Each of the first seven is an `[Action]`: list it in a skill's `Tools` with `Actions.Of(…)`.

## Its tests

`kits/work-repositories-agents/tests` is a lab that tests water samples.

| | |
|---|---|
| What an agent writes down is an item of its own, with where it came from | `what_an_agent_writes_down_is_an_item_of_its_own_with_where_it_came_from` |
| It reads by reference, and only what whoever asked may | `an_agent_reads_the_repository_by_reference_and_only_what_whoever_asked_may` |
| An item is handed to a helper and taken back | `an_item_is_handed_to_a_helper_by_reference_and_taken_back` |

```
osy test kits/work-repositories-agents/tests
```

## Source and package

- Package: `Osysharp.WorkRepositories.Agents` — it arrives by itself in an app that uses
  [`Osysharp.WorkRepositories`](../work-repositories) and [`Osysharp.Agents`](../agents); or
  `use Osysharp.WorkRepositories.Agents@1;`
- Source: [`kits/work-repositories-agents`](.) — `model/Tools.osy`
- Uses [`Osysharp.Assignments.Agents`](../assignments-agents) to know which piece of agent work an item came from.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.Llm;
using Osysharp.Agents;
using Osysharp.Agents.Testing;
using Osysharp.Assignments;
using Osysharp.Assignments.Agents;
using Osysharp.WorkRepositories;
using Osysharp.WorkRepositories.Agents;
using Waterlab;

[Test(Lab)]
void an_item_is_handed_to_a_helper_by_reference_and_taken_back() {
  var a = Opened();
  Guid method = Guid.Empty;
  runas (Cleo) {
    var analysis = Assignment.Single(x => x.Id == a);
    method = AddItem(analysis, new NewItem { Kind = ItemKind.Document, Title = "Method note", Content = "Lead above 10 µg/l fails." }).Id;
    AddItem(analysis, new NewItem { Kind = ItemKind.Input, Title = "Client's letter", Content = "Please hurry." });
  }
  // The reviewer is on no analysis: it reads nothing.
  Assert.Equal(0, ReviewerReads());
  runas (Cleo) { AskTheReviewer(RepositoryItem.Single(i => i.Id == method)); }
  // Handed the method note, it reads the method note — one item, not the repository.
  Assert.Equal(1, ReviewerReads());
  runas (Cleo) { ThankTheReviewer(RepositoryItem.Single(i => i.Id == method)); }
  Assert.Equal(0, ReviewerReads());
  // Only whoever is in charge of the analysis hands its items on.
  runas (Ines) { AddParticipant(Assignment.Single(x => x.Id == a), Hugo, ParticipantRole.Contributor); }
  runas (Hugo) { Assert.Throws<NotAuthorized>(() => AskTheReviewer(RepositoryItem.Single(i => i.Id == method))); }
}
```

From [`tests/findings.test.osy`](tests/findings.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Llm;
using Osysharp.Agents;
using Osysharp.Agents.Testing;
using Osysharp.Assignments;
using Osysharp.Assignments.Agents;
using Osysharp.WorkRepositories;
using Osysharp.WorkRepositories.Agents;
using Waterlab;

[Test(Lab)]
void a_message_an_agent_records_is_correspondence_of_its_own() {
  var a = Opened();
  runas (Cleo) {
    var script = AgentScript.Of(
      Says.Tool(Actions.Of(RecordCorrespondence), Write(a, "Asked the client for a second sample", "Could you send a second sample of W-17?", "Sent by mail")),
      Says.HandBack(new Verdict { Safe = false, Why = "Waiting for a second sample." }));
    ScriptedAnswer.Stub((model, request) => script.Answer(model, request));
    Assert.True(Analyse(W17()).Done);
  }
  Background.Settle();
  runas (Cleo) {
    var analysis = Assignment.Single(x => x.Id == a);
    var letter = ItemsOf(analysis, ItemKind.Correspondence).Single();
    Assert.Equal("Asked the client for a second sample", letter.Title);
    Assert.Equal(Agents.Of<Analyst>().Actor.Id, letter.AddedBy!.Id);
    var v = CurrentOf(letter);
    Assert.Equal("Sent by mail", v.From);
    Assert.Equal("RecordCorrespondence", v.Tool);
    Assert.Equal(WorkOf(analysis, "analyse")!.Id, v.SourceRef);
  }
}
```

From [`tests/findings.test.osy`](tests/findings.test.osy).

## Learn more

- [The Osysharp.WorkRepositories.Agents reference](https://osysharp.com/reference/agent/kit-work-repositories-agents/)
- [Osysharp.WorkRepositories.Agents in the package catalogue](https://osysharp.com/packages/#osysharp-workrepositories-agents)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
