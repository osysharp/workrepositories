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
