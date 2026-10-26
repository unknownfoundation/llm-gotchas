# Testing and evals

Needs: [Generation parameters](../generation-parameters/README.md) · [Prompting](../prompting/README.md) · [Tool calling](../tool-calling/README.md) · [Retrieval and RAG](../retrieval-and-rag/README.md) · Reviewed October 2026

Testing an LLM feature checks the code around the model call and scores the replies the model gives, before release and on live traffic. What a check can show depends on what answers the model call and on what decides a pass.

- **System prompt**: the instructions meant to hold for the whole chat, sent as the first message of the list or, on some APIs, as a separate field.
- **Eval case**: one input with a known good answer, or a rule a good reply must meet; an **eval set** is a collection of them.
- **Grader**: what decides whether each reply passes: code that compares it with the answer, a person, or another model call, often called a judge model.
- **Held-out set**: eval cases kept out of everything that shaped the feature, so that the score speaks for inputs the feature has not seen.
- **Accuracy, precision and recall**: accuracy is the share of all items a check labels right; precision is the share of the items it flags that are real; recall is the share of the real items that it flags.
- **Gold passages**: the passages a person marked as answering each question of a search eval.
- **Precision@k, recall@k and MRR**: three scores for a search, each worked out per question over its first k results and then averaged across the questions: precision@k is the share of those results that are gold, recall@k is the share of the question's gold passages found among them, and MRR is one divided by the rank of the first gold passage.
- **Trace**: the record of every call made for one user request or one agent run: what each call sent, what came back and how long it took.

Every live test and eval run on a provider's API is billed by the token, and once the provider retires a model version, you can no longer repeat a reference run made on it, the run you compare new scores with; on your own server the model's files are yours, so a reference run can be repeated for as long as you keep them, at the cost of time on your hardware. A check works the same way on every kind of system that allows it.

A feature is usually checked in five ways:

| Way | What answers the model call | Where you find it |
| --- | --- | --- |
| Faking the model | a fake model that your test puts in place of the call | your code, on a provider's API or your own server |
| Replaying recordings | replies saved from real calls earlier, played back to your code | your code, on a provider's API or your own server |
| Calling the model live | the real model, once per test, with a check on its reply | a provider's API, your own server or a ready-made app |
| Running an eval | the real model, on every case of an eval set, with a grader turning the replies into a pass rate | a provider's API, your own server or a ready-made app |
| Monitoring live traffic | the real model, on what real users send, with your code recording each call | your code, on a provider's API or your own server |

A public benchmark is an eval set published for comparing models. A 2024 study wrote new problems to match a public maths benchmark: several model families scored lower on the new problems than on the benchmark itself, while many others showed little gap. A model that leads a public benchmark still needs a run on your own cases.

This topic lets you:

- look at a green test, a replayed recording or a rerun in CI and say which change it could have caught
- weigh an eval score, a judge's verdict or a search metric before it settles a choice between prompts, models or rankers
- work back from a quality drop, a slow-reply complaint or an agent's wrong answer to what your logs recorded

###### 1. Testing the language fix with a fake model

At a hotel group, the guest inbox drafts a short reply to every message, and guests who wrote in Portuguese were getting drafts in English. You added a line to the system prompt telling the model to answer in the guest's language, plus a unit test whose fake model is set to return "O seu quarto está confirmado." and which checks that the draft contains "quarto". The test passes, and you close the ticket.

What has the passing test shown?

<details><summary>A. That guests who write in Portuguese get Portuguese drafts</summary>

✗ The test was written for the fix and feeds it a Portuguese message, so a pass reads as the fix at work. No model read that message on any run: the fake returns the Portuguese sentence you gave it, and would return it unchanged if the prompt demanded English.

</details>

<details><summary>B. That your code turns the reply into the draft</summary>

✓ Right. It has shown that your code passes on the text it gets back from the call, and nothing about how a model treats the prompt. The test goes green with the line, without it and with no system prompt at all, since the fake never reads what it is sent.

When a merge drops the line a week later, the build stays green and Portuguese guests start receiving English drafts again. The front-desk staff who approve each draft are back to rewriting replies by hand, and the green test sends the engineer who takes their complaint looking everywhere except the prompt.

Faking the model puts a fake model where the call was, and this fake returns its fixed sentence whatever arrives, so the assertion looks for "quarto" in the very sentence you typed into the fake. Tests built on a fake can settle two things about your code: what it sends, and how it handles a reply. To protect the line, let the fake keep a copy of its request and assert that the system prompt includes it; that version goes red on the day of the merge. Whether a model obeys the line is for running an eval, across many guest messages in several languages. The fake stands in before any request leaves your code, whether the real call would go to a provider's API or to your own server.

</details>

<details><summary>C. That the system prompt still carries the line</summary>

✗ A fake sitting exactly where the call was seems well placed to notice what the call carries. This one never inspects it: the messages go in and the fixed sentence comes out. Only an assertion on the recorded request can catch a missing line, and this test makes its assertion on the draft alone.

</details>

<details><summary>D. Nothing about your code or the drafts</summary>

✗ After a green test has once hidden a real bug, writing off fakes entirely is an understandable reaction. This one does check something: that your code turns the reply into the draft unchanged. A broken return path, such as a draft cut to its first word, would turn it red.

</details>

---

###### 2. Replaying last month's recorded replies

The integration tests of a ticket summariser replay replies that were recorded from the provider's API last month. CI runs them on every change, nobody has recorded the replies again since, and the team counts on them to warn of anything that breaks the summaries.

Which of these changes turns one of the integration tests red?

<details><summary>A. An outage at the provider during the run</summary>

✗ Integration tests are where outages usually surface, since they are the tests that reach other systems. These reach a file: each reply comes from last month's recording, so the provider can be down for the whole run and every check still passes on the old text.

</details>

<details><summary>B. The provider renaming a field in its responses</summary>

✗ A renamed field breaks parsing, and parsing is exactly what these tests exercise, which makes this the change they should catch. The reply they parse is the recorded one, in last month's format. The new field name exists only in the live API's responses, and the run never fetches one.

</details>

<details><summary>C. A timeout too short for the real API's replies</summary>

✗ These tests go through the same client code as production, timeout included, so a tight timeout looks testable here. A replayed reply comes back from a file in a moment, nowhere near the limit. The timeout fails only against the real API, where a long reply takes seconds.

</details>

<details><summary>D. A bug in your code that reads replies</summary>

✓ Right. Only the bug turns a test red: the replay still hands over last month's reply, and code that now misreads it fails the check.

If the provider renames a field next month, every recorded test still passes, and the support lead finds the morning queue full of tickets with blank summaries before any build has gone red.

Replaying recordings plays back replies saved from real calls earlier, so everything on the far side of the call is frozen on the day of recording: the model, the API's response format and how long a reply takes. A replayed reply arrives at once, a renamed field never appears in it, and an outage never reaches the run. What changes on your side of the call is still tested against it: the code that reads the reply, parses it and acts on it. A prompt edit depends on the tool: one that looks up a recording by the request's URL alone plays the old reply back for the new prompt, and one that matches the whole request finds no recording and fails; neither shows how a model answers the new prompt. Record again on a schedule and after every prompt change, and keep a few live tests for what the far side does today.

</details>

---

###### 3. Giving failed live tests two more tries in CI

The chatbot team at a bank has live tests that sometimes fail with nothing changed, so CI reruns them: a failed test gets up to two more tries, and the build goes green if any try passes. For months the test of refund answers failed 2% of its tries. Since this morning's prompt change it fails 30% of them, and each try passes or fails on its own, whatever the last one did.

What colour does the build come out from now on?

<details><summary>A. Red on every build</summary>

✗ A prompt change is code, and broken code fails every time, so you expect the build to turn red and stay red. This failure comes and goes: the test still passes 70% of its tries, and with three tries per build at least one passes on 97.3% of builds.

</details>

<details><summary>B. Red on most builds</summary>

✗ Three tries give the fault three chances to show, the way three inspections give a defect three chances to be spotted. The build counts the other way round: it is red only when every try fails, and at 70% a try, three failures in a row come up on 2.7% of builds, about one in 37.

</details>

<details><summary>C. Green on almost every build</summary>

✓ Right. The build goes red only when all three tries fail, on about one build in 37; the other 97.3% stay green, though the refund test now fails three tries in ten.

Customers get one try each, so a customer who asks what this test asks is now given a wrong refund answer three times in ten, and the build page stays green. The rare red build looks like the flakiness the reruns were added to absorb, and someone presses rerun.

Calling the model live means each reply can differ from the last, and a rerun policy keeps the best of three tries. A try that fails 30% of the time fails three times running on 2.7% of builds. Before the change, at 2% a try, a red build came about once in 125,000. A green build keeps no note of how many tries it needed, and a genuine regression is granted the same three chances as a dropped connection. Keep every try's result, and raise the alarm on each test's pass rate over its tries, whatever colour the build ends up.

</details>

<details><summary>D. Red on about 30% of all builds</summary>

✗ Before the reruns, a test that failed 30% of its tries turned about that share of builds red, and you carry that habit over. The reruns change the count: a build is red only when all three tries fail, which is far rarer than one failure.

</details>

---

###### 4. Choosing between two instructions on 100 cases

On a recipe site, the substitution helper suggests swaps for missing ingredients. Two candidate instructions have each had a single run against the same 100 eval cases, with code grading every reply pass or fail, and the product manager plans to switch to the higher scorer tomorrow morning.

| Instruction | Cases passed | Cases run |
| --- | --- | --- |
| A | 61 | 100 |
| B | 69 | 100 |

What do these scores tell the product manager?

<details><summary>A. That B is the better of the two instructions</summary>

✗ Eight more passes on the very same cases is the kind of margin a launch slide is built on. Run each instruction again and both totals move: at 100 cases a pass rate can sit anywhere in a band about 18 points wide, and two equally good instructions end up eight or more points apart in about a quarter of such comparisons.

</details>

<details><summary>B. That it is still open which one is better</summary>

✓ Right. Neither instruction has won. One run over 100 cases places each pass rate only within a 95% range about 18 points wide. That range is 51% to 70% for A and 59% to 77% for B, and the two share everything from 59% to 70%.

The product manager tells the team that B is eight points better. If next month's run puts A in front, nobody can say whether the helper got worse or the first gap was luck, and the switch has to be argued all over again.

Running an eval gives a pass rate for one run over one sample of cases: draw another 100 cases from the same stream of requests, or run these again, and the rate lands somewhere else. Treat the two runs as separate samples and the true gap behind the 8 points could be anywhere from about −5 to 21. Even if the two instructions were equally good, 26.6% of pairs of runs would put them 8 or more points apart, one way or the other. Both instructions saw the same cases, though, so the cases where one passed and the other failed tell you more than the totals, which hide them. If B won all 8 of those, chance alone would give so lopsided a split 0.8% of the time; if B won 21 and A won 13, chance would give it 22.9% of the time. Read the case-by-case results, repeat each run a few times, and grow the set until the range is narrower than the smallest gap that would change your decision.

</details>

<details><summary>C. That the two instructions are equally good</summary>

✗ When a test fails to separate two things, the habit is to file them as the same. These counts leave both stories open: the gap of B over A could be anywhere from about −5 to 21 points, wide enough to hold a dead heat and a clear lead for B alike.

</details>

---

###### 5. Drawing a router's examples from its eval file

A mobile network routes customer emails to the billing, coverage or roaming queue. A file holds 400 past emails, each labelled with its right queue. For each email to route, `nearest` searches that file for the three emails most alike to it, and your code puts them into the prompt as examples. The eval routes all 400 emails of that same file and checks each answer against its label. `call_model` wraps the provider's client library and returns the reply as plain text. The eval reads 96%, against 79% with three fixed examples, and the head of operations wants the number in this week's update.

```python
labelled = read_rows("labelled_emails.csv")

def prompt_for(email):
    shown = nearest(email, labelled, n=3)
    lines = [f"Email: {row['body']}\nQueue: {row['queue']}" for row in shown]
    return "\n\n".join(lines + [f"Email: {email}\nQueue:"])

hits = 0
for row in labelled:
    answer = call_model(messages=[{"role": "user", "content": prompt_for(row["body"])}])
    hits += answer.strip() == row["queue"]
print(hits / len(labelled))
```

What does the 96% measure?

<details><summary>A. How well it copies a queue shown in its prompt</summary>

✓ Right. Every one of the 400 prompts shows the model the email it is about to route, with the right queue written under it, ahead of two other examples. The file the search runs over is the file being scored, row for row.

Real emails never come with their queue written beside them, so the 96% belongs to a task the router never faces. The head of operations would publish a figure that next month's report on live emails cannot be lined up against.

A held-out set stays out of everything that shaped the feature, and these eval cases are the very pool the examples are drawn from. Nothing is more alike to an email than the email itself, which scores the top of the similarity scale, so the first example is always the email being scored, with its queue, and the model is marked on copying a queue it read a few lines earlier. Keep the emails the examples are drawn from apart from the emails you score, with no email in both, or leave out the scored email itself when its examples are searched; then set the result beside the 79% measured on the same held-out set. Tuning a prompt on the very cases you report gets there more slowly: the cases mould the prompt, and the score no longer speaks for new emails.

</details>

<details><summary>B. How well it routes emails it has never seen</summary>

✗ The eval runs the production code end to end on real labelled emails, which is what a careful test is meant to do. The catch sits in the search: the email being scored is a row of the file it searches, so it comes back as its own first example, queue and all. A customer's email never arrives with its queue attached.

</details>

<details><summary>C. How well it routes new emails, a point or two high</summary>

✗ Leaks you have met before were a few duplicated rows, worth a point or two, and this one gets filed with them. Here the leak is built into the search: nothing in the file matches an email as closely as the email itself, so all 400 prompts carry their own answer.

</details>

<details><summary>D. How much similar examples add over fixed ones</summary>

✗ The 17-point rise came with the move to similar examples, so you credit the move. The two runs test different tasks: 79% with examples that never contain the case, 96% with prompts that hand over its answer. The difference blends the move with the leak.

</details>

---

###### 6. Letting a judge model pick the better answer

Before a cooking app ships a new prompt for its assistant, its team compares old and new answers to 300 questions with a judge model, which reads both answers and names the better one. The new prompt's answers run about twice as long, and the judge picks them in 70% of pairs. You then add an instruction asking the old prompt for answers as detailed as the new ones, and run the judge again.

When the judge runs again, what should you expect from the new prompt's share of wins?

<details><summary>A. Stay near 70%</summary>

✗ A judge is told to pick the better answer, and better means content to you, so you expect its verdict to ignore length. A 2024 study found the opposite: a judge's preference moved a long way with length alone, and matching the lengths takes away the head start the new answers had.

</details>

<details><summary>B. Rise above 70%</summary>

✗ Padding is a known sin in writing, so a judge might be expected to punish answers stretched out on request. The 2024 study saw the reverse: when one model was told to give as much detail as possible, its answers won far more often against its own standard ones.

</details>

<details><summary>C. Fall below 70%</summary>

✓ Right. Expect the share to fall. A 2024 study found an automatic judge favouring longer answers, and the old prompt's answers now come close to the new ones in length, so part of the new prompt's lead goes with that difference.

Had the team shipped on the 70%, home cooks would scroll through answers twice as long on a phone propped by the stove, for a lead the judge may have handed out for length.

A judge model is a grader that writes its verdict from the two answers it reads, and their length is part of that input like everything else. The study behind Length-Controlled AlpacaEval found an automatic judge preferring longer outputs: one model, compared with its own standard answers, went from winning 22.9% of comparisons to winning 64.3% with nothing changed but an instruction to be brief or detailed. The study corrects scores for length, and the same idea works on your own judge: compare answers of matched length, and have a person check the judge's verdicts on a sample before you trust them. Those were the models of a 2024 study, so measure yours.

</details>

---

###### 7. Reading a 99% score for an injection detector

An agent that answers a utility company's customers reads every incoming message, and a detector screens each one first for prompt injection, text meant to override what the agent was told to do. Tested on 5,000 labelled messages, 50 of them injections, the detector gets 99.0% accuracy. The head of security wants to know how many of the 50 it stopped.

What can you tell the head of security from the 99.0%?

<details><summary>A. That it stopped about 49 of the 50</summary>

✗ The question on the table is how many it stopped, and 99% of 50, about 49, answers exactly that. This figure counts every decision on all 5,000 messages, and 4,950 of them are harmless ones that a detector gets right by staying quiet. Never raising a flag at all already earns 99.0%.

</details>

<details><summary>B. That it stopped at least half of them</summary>

✗ It is hard to picture a detector this close to perfect missing most of what it was built for. The 99.0% allows it: a detector that stays silent on every message lets all 50 injections through and still reaches the figure.

</details>

<details><summary>C. That it stopped none of them</summary>

✗ The do-nothing detector is the textbook case of a misleading accuracy, so it is the first one that comes to mind. It is one of many that score 99.0% here: stopping all 50 injections at the cost of 50 false alarms also makes 50 mistakes and earns the identical score.

</details>

<details><summary>D. That it stopped anywhere from 0 to 50</summary>

✓ Right. The 99.0% fits a detector that stops none of the 50, one that stops all of them, and every count between. Staying silent scores 99.0% on the 4,950 harmless messages alone; stopping every injection while flagging 50 harmless messages leaves 50 mistakes, the same count, and the same score.

With 99% in the rollout note, the detector goes live. Should it be the one that never flags, every injection reaches the agent untouched, and the head of security first hears of it from a screenshot of the agent following a stranger's instructions.

Accuracy is the share of all items a check labels right, and here 99 messages in 100 are harmless, so they carry almost the whole score and the 50 injections can move it by a point at most. Precision and recall are built from the flags and the injections only, so the quiet majority cannot prop them up. The never-flagging detector has a recall of 0; stopping half with 25 false alarms gives a precision of 0.5 and a recall of 0.5; stopping all 50 with 50 false alarms gives a recall of 1.0 at a precision of 0.5. Each of the three scores 99.0% accuracy. Before signing off, ask how many of the injections were flagged and how many of the flags were injections, as counts.

</details>

---

###### 8. Adding gold passages to a search eval

An insurer's claims assistant is checked by a search eval of 150 questions. Last quarter each question had a single gold passage, and recall@5 stood at 0.84. Since then an adjuster has gone back through the policy wordings and marked every passage that holds part of an answer, so each question now has two to four gold passages, its original one among them. The search, its index and the questions are as they were, and recall@5 now reads 0.51.

What share of the 150 questions now have at least one gold passage in their top 5 results?

<details><summary>A. At least last quarter's 84%</summary>

✓ Right. At least 84%. The 126 questions that found their one gold passage last quarter still find it, since the search returns what it returned then, and the newly marked passages can only raise that count.

The engineer who owns the search sees 0.51 on the quarterly report and spends a week looking for a fault that is not there. The figure now counts how much of each question's answer turns up in the top 5, a different matter from whether any of it does.

Recall@k averages, over the questions, the fraction of each question's gold passages that appear in its top k. While every question had one gold passage, that fraction could only be 0 or 1, so the average was the share of questions that found gold. Once questions have several, a partial find counts as a fraction: a question that finds one of its three gold passages adds 0.33, though it did find one, and the average falls below the share with a hit. Put the share with any gold passage beside recall@k, add the share that found every passage when an answer needs all of them, and when the gold set changes, score last quarter's search against the new set before comparing quarters.

</details>

<details><summary>B. About 51%</summary>

✗ For a whole quarter recall@5 was the share of questions that found their passage, each having only one, and the dashboard is still read that way. With two to four gold passages each, a question now adds the fraction of them it found: one passage out of three counts as 0.33, yet that question found gold.

</details>

<details><summary>C. Between 51% and last quarter's 84%</summary>

✗ A drop that large reads like a search that kept some answers and lost others. The search returns what it returned last quarter, so every question that found its original passage still finds it, and a newly marked passage can only add questions with a hit.

</details>

---

###### 9. Adding a reranker to a tax help bot

The search behind a tax help bot returns 6 passages per question, and only the first 2 of them go into the prompt. A dashboard follows precision@6, recall@6 and MRR across 120 test questions with one gold passage each, and every gold passage used to rank first. You ship a reranker, a model that reorders the returned passages: the same six come back for every question, and each gold passage now ranks third.

Which of the dashboard's numbers change?

<details><summary>A. No number on the dashboard moves</summary>

✗ The reranker added nothing and removed nothing, and a change that only reorders a list should leave every number about it alone. Two of the numbers only count gold passages among the six and stay put. MRR is read off the first gold passage's rank, and falls from 1.00 to 0.33 when that passage moves from first to third.

</details>

<details><summary>B. Each of the three numbers moves</summary>

✗ The three numbers share one dashboard panel and have moved together whenever the search changed before, so a reshuffle this big should show in each. Precision@6 and recall@6 only count gold among the six: one in six gives 0.17 and 1.00, before the reranker and after it. MRR alone, from 1.00 to 0.33, sees the move.

</details>

<details><summary>C. MRR falls, the other two stay put</summary>

✓ Right. MRR drops from 1.00 to 0.33, while precision@6 stays at 0.17 and recall@6 at 1.00: the gold passage is still in the six, now third.

Only the top 2 reach the prompt, so every test question has lost its gold passage. Two of the dashboard's three lines stay flat that week, and the first complaint comes from a freelancer who was quoted the wrong deduction limit.

Precision@k and recall@k count gold passages within the top k and pay no attention to their order inside it; MRR turns the first gold passage's rank into its reciprocal, 1/1 before the reranker and 1/3 after. A score taken at 6 judges the six passages returned, while the model only sees two. Watch a number that cares about rank, such as MRR, and set k to the number of passages the prompt receives: here recall@2 falls from 1.00 to 0.

</details>

<details><summary>D. Recall@6 and MRR fall, while precision@6 stays put</summary>

✗ The gold passage has dropped out of the prompt, and recall is where you would expect a lost answer to register. Recall@6 asks only whether the passage is among the six returned, and it is, in third place. The loss sits between the six a score inspects and the two the model is given.

</details>

---

###### 10. A nightly eval that drops in a week with no deploy

A logistics company runs its support classifier through a provider's API. Every night an eval sends 2,000 fixed cases straight to the model, and code grades each reply against a fixed answer. On Wednesday night the score read 88%; on Thursday night it read 81%. No one shipped anything that week, and every call's log line, before and after, shows the model name from your config.

What is the likeliest cause of the drop?

<details><summary>A. Noise between runs of the eval</summary>

✗ Every nightly chart wobbles, and with the config and the code untouched, luck is what remains to blame. Luck on 2,000 cases is small: on 2,000 cases a rate near 88% wanders by a point or so, and a fall to 81% by luck alone would practically never happen.

</details>

<details><summary>B. A change behind the model name</summary>

✓ Right. Something on the provider's side changed behind the name your code sends: a newer release now answering to it, or a change in how the provider serves it. The log looks identical either way, as it records the name you asked for.

The engineer on call spends Thursday combing a deploy history with nothing in it, then the eval's own code, then its cases. Every call log line, before and after the drop, names the model in the config, and that line is what keeps the provider off the list of suspects.

A nightly eval is running an eval on a schedule, and its log, like any record of live traffic, holds what your code sent: the model name in a request is what you asked for, which need not be what answered. Some names stand for one fixed release; others are labels the provider moves to each new release, and even a fixed release runs on serving systems the provider updates, which one maker says can shift behaviour slightly. The drop is no accident of sampling: on 2,000 cases, 88% has a 95% range of 86.5% to 89.4%, and 81% one of 79.2% to 82.7%. Name a fixed release in the config where the provider offers one, log the release the response reports where the API returns it, and keep the provider's change notes beside your eval history. On your own server the files change only when you change them, so the same drop there points at your own change: new model files, a new server version or new settings.

</details>

<details><summary>C. The model changing as it learns from the traffic it serves</summary>

✗ Chat products are said to get worse the more people use them, and a nightly score seems to show it. A model's weights are fixed numbers while it serves requests: no call, yours or a customer's, writes to them. Served unchanged, they give tonight the same spread of replies as last month, so a change has to come from outside them.

</details>

<details><summary>D. The fixed cases going stale</summary>

✗ Eval sets do age as products and customers change, and this one has not been touched in months. Ageing makes a set less like real traffic without moving its score: each case has a fixed answer graded by code, so an unchanged model gives these cases the same spread of results year after year.

</details>

---

###### 11. Promising replies in about six seconds

Travellers on a booking site use its assistant for two kinds of request: about 85 in 100 are quick questions, and the rest ask for a full itinerary, which takes many times longer to write. The dashboard reports a mean reply time of 6.2 seconds for the day, and marketing starts promising replies in about six seconds.

What does the mean say about how long that day's travellers waited?

<details><summary>A. Most waited roughly the 6.2 seconds shown</summary>

✗ An average is taught as the typical case, and for one kind of request it nearly is. This day mixes two kinds, and the slow one lifts the mean far above the quick one: picture quick answers at about two seconds and itineraries at about thirty, and the mean comes out at 6.2 seconds, where hardly any reply lands.

</details>

<details><summary>B. Half waited longer than six seconds</summary>

✗ Half on each side is how the middle of a set behaves, and the mean sits in the middle when times spread evenly around it. Here a minority of slow replies breaks that: the itineraries, 15 in 100, lift the mean above the quick answers, so about 85 replies in 100 come in under it.

</details>

<details><summary>C. Most waited longer than six seconds</summary>

✗ You have seen a mean hide a slow tail, and the tail you remember meant most people waiting longer than the dashboard said. The tail here is the itineraries, 15 requests in 100. The other 85 are quick questions, and the mean sits above them, so most travellers waited less than six seconds.

</details>

<details><summary>D. Most waited well under six seconds</summary>

✓ Right. Most waited well under six seconds. The quick questions are 85 in 100, and the itineraries, many times slower, pull the mean above them. In a picture of 20,000 such replies, with quick answers near two seconds and itineraries near thirty, the median is 2.1 seconds and not one reply falls between 5.5 and 6.5 seconds.

Travellers with a quick question get their answer well inside the promise, while those planning a trip wait several times longer than promised. The one-star reviews come from the trip planners, and the support desk, holding a single figure of 6.2, cannot even tell them how long an itinerary usually takes.

A mean adds up every reply time and shares the total out evenly, so a few long replies raise it for everyone: in the picture, 15 itineraries in 100 at about thirty seconds lift it from two seconds to 6.2, while 85% of replies finish sooner and the 95th percentile is 31.7 seconds. A percentile tells you how long a given share of travellers waited: the median for the one in the middle, p95 for the slowest one in twenty. When monitoring live traffic, chart the median and p95, or give each kind of request its own line, and promise a time per kind.

</details>

---

###### 12. Keeping a trace of an agent's steps

A courier's support agent answers delivery questions with a lookup tool, and your code keeps a trace of what each step sent. `call_model` returns the model's reply as one message, which holds a tool request when the model wants a lookup, and `save_trace` writes the trace to a JSON file when the run ends. One run took four steps and gave a customer the wrong delivery day, and you want to know what the model was sent at step 1.

```python
def run_agent(task):
    trace = []
    messages = [{"role": "system", "content": AGENT_PROMPT},
                {"role": "user", "content": task}]
    for step in range(1, 11):
        trace.append({"step": step, "sent": messages})
        reply = call_model(messages=messages)
        messages.append(reply)
        if not reply.get("tool_request"):
            break
        messages.append({"role": "user", "content": run_tool(reply["tool_request"])})
    save_trace(trace)
    return reply["content"]
```

What does the trace show under step 1?

<details><summary>A. Every message of the run, final reply included</summary>

✓ Right. Under step 1 the trace shows all nine messages of the run, the wrong answer at the end included, and so do steps 2, 3 and 4. The four entries hold one list, and it was saved after the run had finished growing it.

You set out to see what the model knew when it chose its first lookup, and the file cannot tell you: each step appears to have read the whole run, its own answer included, so the hunt for the wrong delivery day starts from a record of a request no step ever sent.

A trace records what each call sent only if each entry is its own copy. `trace.append({"step": step, "sent": messages})` stores the `messages` list itself, and the loop keeps appending to that one list, so when `save_trace` writes the file, all four entries show it in its final state, nine messages long. The JSON writer writes one list out several times without complaint; only a list that contains itself would make it fail. Store a copy at each step, such as `copy.deepcopy(messages)`, and the entries come out at 2, 4, 6 and 8 messages, which is what the model read; better still, log the request as it leaves your wrapper.

</details>

<details><summary>B. The system prompt and the customer's question</summary>

✗ Appending at the top of each step feels like taking a snapshot of that moment, and a log entry is meant to be one. The entry holds the list itself, not its contents at that moment. The loop kept appending to that one list, and the file was written after the last reply.

</details>

<details><summary>C. Every message except the final reply</summary>

✗ Once you see that the entries share one list, the last model call is the natural place to stop reading it. The list grew past that call: the final reply is appended before the loop breaks, and the file is written after that. So each entry, step 1 included, ends with the wrong answer.

</details>

<details><summary>D. No step 1 entry, only one for step 4</summary>

✗ Writing the same object four times is how you would overwrite one record, leaving only the last. The trace is a list of four separate entries, each with its own step number. Each points at the same message list, so all four come out showing the whole run.

</details>

---

## Further reading

- [Length-Controlled AlpacaEval](https://arxiv.org/abs/2404.04475), 2024: an automatic judge that favoured longer answers, and a correction that compares two models as if their answers were the same length.
- [A Careful Examination of Large Language Model Performance on Grade School Arithmetic](https://arxiv.org/abs/2405.00332), 2024: a new set of maths problems written to match a public benchmark, on which several model families lost up to 8% accuracy while many models, frontier ones especially, showed little sign of overfitting.
- [Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena](https://arxiv.org/abs/2306.05685), 2023: strong judge models agreed with human preferences over 80% of the time, about as often as people agree with each other, and the paper examines their position, length and self-preference biases.

Next: [Releasing LLM changes](../releasing-llm-changes/README.md)
