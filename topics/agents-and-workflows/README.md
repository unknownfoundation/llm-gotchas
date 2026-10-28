# Agents and workflows

Prev: [Memory](../memory/README.md) · Next: [Security](../security/README.md) · Reviewed October 2026

An agent is a loop in which the model picks each next step, another tool call or the answer; a workflow runs steps your code fixes in advance, and the model does the work inside each. Either way a run is a series of separate model calls, and your code decides what each call is sent.

- **Context window**: the most tokens a model can handle in one call, counting both what it reads and what it writes.
- **Tool**: a function your code offers the model; a reply can ask for it, and your code runs it and sends back the result.
- **Step**: one round of a run: in an agent loop, one call whose reply either ends the run or leads to the next call; in a workflow, one of the stages your code runs in order, which can hold a call and code around it.
- **Check pass**: one extra call that reviews an answer before it is used, and keeps or corrects it.
- **Checkpoint**: a saved copy of a run's state that the run can be resumed from: the messages so far and anything your code records beside them, such as an order number or a manager's answer; it lasts as long as the place your code keeps it, so one in a process's memory ends with that process and one in a file or a database outlives a restart.
- **Approval**, often called human in the loop: a pause in which a run waits for a person's answer, then resumes once the answer arrives.

A provider's API and your own server answer each call from the messages they are sent, so on both the loop, the state and the checkpoints live in your code; a ready-made app such as a coding assistant runs an agent loop of its own and can stop to ask you before it edits a file or runs a command.

Most systems combine a few ways of wiring calls together:

| Pattern | What it does | Who picks the next step |
| --- | --- | --- |
| Chain | runs fixed steps in order, each working on the previous step's output | your code |
| Router | one call reads the input and names a branch, and your code hands the work to that branch | the router call, once per input |
| Parallel workers and a merge | several calls work on parts of one task at the same time, and a last step, the merge, combines their results | your code |
| Orchestrator and workers | a lead, the orchestrator, reads the task and decides at run time how to split it; worker calls do the parts, and the lead combines what they return | the lead, once per task |
| Evaluator loop | a writer call drafts, a reviewer call (the evaluator) judges the draft, and the writer revises it in light of the review | your code, from the reviewer's verdict |
| Voting | sends one call several times and keeps the answer that most of the runs return | your code |
| Agent loop | the model calls tools one step after another and answers when it decides it is done | the model |

This topic lets you:

- decide whether a job needs an agent or a fixed workflow, and work out before launch what extra calls do to the share of answers that come out right
- say what each call in a run of several is sent and gets back, and what ends a loop and what it hands back when it ends
- build a run that stops to wait for a manager's approval and carries on once the answer arrives

###### 1. Two teams that each want an agent

Two teams at a building supplies wholesaler each ask you for an agent. Accounts wants supplier invoices handled: each one is read, matched against its purchase order and posted to accounting. Sales wants answers to open questions about competitors, such as whether a rival now delivers to a new region, the kind of question a person settles by following leads from one page to the next.

How should you build the two jobs?

<details><summary>A. A workflow for the invoices, parallel searches and a merge for the research</summary>

✗ Running the searches side by side gets answers sooner, and a merge step collects what each one found. Your code, though, has to pick those searches before any page is read, and the next lead depends on what the last page said. A search running beside the others never sees that page, so it cannot follow the lead.

</details>

<details><summary>B. An orchestrator with workers for the invoices, an agent for the research</summary>

✗ Giving each part of a job to its own worker is how bigger systems are often drawn, and an invoice has three parts. Those parts are known in advance and always run in the same order. A lead that decides the split at run time can choose a different one on every invoice, so the calls vary and nothing in your code makes sure matching comes before posting.

</details>

<details><summary>C. An agent for the invoices, a workflow for the research</summary>

✗ The invoices carry money and touch three systems, and the job with the most at stake is the one teams give their most capable build. Care with money points to the fixed build instead: the three steps are known in advance, and a workflow runs them the same way every time, while an agent would let the model pick each step, and no line of your code would stop it from skipping the match or posting twice.

</details>

<details><summary>D. An agent for the research, a workflow for the invoices</summary>

✓ Right. The research questions run as an agent and the invoices as a workflow: only the research needs the model to decide its next step from what the last one found. Built as an agent, the invoice job could make a different number of calls on each invoice, with no way to price a month in advance or to show an auditor that every invoice was matched before it was posted.

In an agent loop the model chooses each next step, and that choice earns its cost when the number and order of steps depend on what the results show, as with a search that may or may not find the answer. A workflow fixes the steps and their order in your code. The model still does the reading inside each step, such as pulling the amounts out of an invoice, but it never decides what comes next, so every invoice takes the same calls in the same order and can be tested and priced in advance.

Build the invoices as a workflow; a workflow can still hand a single open-ended step to an agent.

</details>

---

###### 2. Five prompts, each tested alone, answering customer emails

An energy supplier answers customer emails with a chain of five steps, each its own prompt: sort the email, look up the account, find the cause, write the reply, check its tone. Scored alone on 200 emails each before launch, the steps got 198, 196, 190, 194 and 192 right. A mistake at any step spoils the reply the customer gets. Assume each step's mistakes fall on emails independently of the others'.

Compared with the steps' own scores, how often should you expect a customer to get a right reply?

<details><summary>A. As often as the weakest step is right</summary>

✗ The saying about a chain and its weakest link fits steel: one load runs through every link, and the weakest gives first. Here each step fails on its own set of emails. The step that finds the cause spoils 10 emails in 200, the other four spoil more on top of those, and the share of right replies drops below that step's 95 in 100.

</details>

<details><summary>B. Less often than the weakest step is right</summary>

✓ Right. About 86 customers in 100 get a right reply, while the weakest step alone, the one that finds the cause, is right on 95 in 100, so roughly one reply in seven goes out wrong with no single prompt to blame.

A chain feeds each step's output to the next, so a reply is right only on emails where all five steps were right. With independent mistakes that share is the five rates multiplied: 0.99 × 0.98 × 0.95 × 0.97 × 0.96 comes to about 0.86. Each extra step multiplies in another number under 1, so a chain of five steps at 95 in 100 each lands near 77, and ten of them near 60. When mistakes cluster on the same hard emails the chain climbs back toward its weakest step, but it never passes it.

Test the chain end to end on one set of emails, and have your code validate each step's output, retrying or stopping an email that fails.

</details>

<details><summary>C. As often as the average step is right</summary>

✗ The five scores sit side by side in the test report, and their average, 97 in 100, makes a tidy single figure for the whole flow. Each score, though, came from a step tested on its own. A customer's reply needs all five right on the same email, so the rates multiply, and the chain lands near 86 in 100.

</details>

---

###### 3. Handing a refund question to the billing agent

A shop that sells phones runs a support chat that answers customers with one model and sends billing questions to a billing agent run by another team. `call_model` is your thin wrapper around the model call: it takes the messages and returns the reply text, and `billing_agent.ask` is the billing team's client. This chat has never reached the billing agent before. A customer gave the order number in the first of ten messages about a phone that will not charge, and now writes "Then I want a refund."

```python
def handle(chat, text):
    chat.append({"role": "user", "content": text})
    label = call_model(messages=[{"role": "system", "content": ROUTER_PROMPT}, *chat[-6:]])
    if label.strip().lower() == "billing":
        reply = billing_agent.ask(text)
    else:
        reply = call_model(messages=[{"role": "system", "content": SUPPORT_PROMPT}, *chat])
    chat.append({"role": "assistant", "content": reply})
    return reply
```

The router answers `Billing`. Besides its own instructions, what does the billing agent's model call carry?

<details><summary>A. The whole chat so far</summary>

✗ In a chat app, whoever picks up a conversation can scroll back through everything above, and the billing agent picks up this conversation. This code makes a narrower choice of its own: `ask` receives `text` and nothing else, and the ten messages about the phone stay in `chat`, which only the router and the support model read.

</details>

<details><summary>B. The six messages the router read</summary>

✗ The router has read the last six messages, and the agent it picks gets the router's decision. The router's reply is one word, and your code uses it only to choose the branch. The six messages go nowhere else, and they would not have carried the order number anyway, since it sits in the chat's first message.

</details>

<details><summary>C. The refund message alone</summary>

✓ Right. The billing agent's call carries its own instructions and "Then I want a refund.", and nothing else: no phone, no fault and no order number, so whatever it replies starts from nothing ten messages into the chat.

A router names a branch, and your code hands the work to that branch; what the chosen agent knows is whatever your code sends along. Each call reads only its own request, and the billing agent has never seen this chat, so it knows nothing of it unless your code passes it on.

Send the billing agent what it needs: the chat itself, or the facts the chat settled, such as the order number and the fault, written into the message.

</details>

<details><summary>D. The refund message with a short summary of the chat</summary>

✗ A colleague handing over a case writes a short note on where things stand, and the router hands the case over. No line of this code writes one: the router's only output is a label, and `ask` gets `text`, the refund message exactly as the customer typed it.

</details>

---

###### 4. Merging four contract notes into one report

At a manufacturer, the procurement team reviews each supplier contract with a tool that runs four worker calls side by side, each reading the whole contract for one topic: payment, liability, termination and data. A merge call then gets the four notes and an instruction to write one report. On one contract the payment note says invoices are due in 30 days, and the termination note, citing a late-payment clause, says 60.

When it writes the report, what can the merge call do about the 30 days and the 60?

<details><summary>A. Flag the clash and leave the term open</summary>

✓ Right. The merge call can flag the clash and leave the term open: it has both terms in front of it, 30 days in one note and 60 in another, with no way to tell which one the contract says, so whatever term the report gives comes from one of the two notes, not from the contract.

With parallel workers and a merge, the last step combines what the workers return, and it reads only its own request: here the four notes and the instruction, and not one line of the contract. It can line the notes up, drop repeats and notice where they disagree, since both figures sit in its input. It cannot check a note against a clause it never received, and each worker's call ended when its note came back, so there is no one left to ask.

Have each worker quote the clause behind every figure, so the merge has the words to compare, or send a disagreement back to a call that reads the contract, or to a person.

</details>

<details><summary>B. Check both figures against the contract's clauses</summary>

✗ The merge call writes the final report, and a person who writes a report on a contract checks it against the contract. This call holds the workers' notes and its instruction, nothing more. The contract went to the workers, and none of its text reaches the merge, which leaves no clause to check either figure against.

</details>

<details><summary>C. Ask the two workers again which term is right</summary>

✗ Agents are described as working in a team, and team members stay on hand for questions. Each worker was one call, and it ended when its note came back. The merge call can send nothing anywhere: it writes one reply, and only your code could start a new call to settle the point.

</details>

<details><summary>D. Settle it with the payment worker's 30 days</summary>

✗ The payment worker is the specialist on payment terms, and on payment terms the specialist's note is the one teams usually trust. Both notes are claims about the same contract, and who wrote a note says nothing about what the clause says: the termination worker cites a late-payment clause the payment worker may have passed over, so picking 30 would only hide the clash in the report.

</details>

---

###### 5. Gathering pricing, legal and tech research for one report

You maintain a consultancy's market-entry tool. It starts three workers side by side, on pricing, legal rules and tech, and a last call writes the report from their findings. `acall_model` is your async wrapper around one model call; it returns the reply as text, and `asyncio.gather` runs the three workers at the same time. In one run the replies came back legal first, at 2 seconds, then tech at 4 and pricing at 7.

```python
async def research(question):
    state = {"question": question}

    async def worker(prompt):
        reply = await acall_model(messages=[{"role": "user", "content": prompt + question}])
        state["result"] = reply

    await asyncio.gather(worker(PRICING), worker(LEGAL), worker(TECH))
    return await acall_model(messages=[{"role": "user", "content": "Summarise: " + state["result"]}])
```

Which findings reach the report call?

<details><summary>A. The findings of all three workers</summary>

✗ Every worker saves what it found, three saves in all. This `state` is an ordinary dictionary, and `state["result"]` holds one value at a time. Every worker assigns to that one key, each assignment overwrites the one before, and the report call reads whichever came last.

</details>

<details><summary>B. The tech findings only</summary>

✗ Tech is the last call in the `gather` line, and `gather` hands its results back in the order the calls were passed. These workers hand nothing back: each writes to `state` the moment its own call returns. Legal writes at 2 seconds, tech at 4 and pricing at 7, and the write that stays is pricing's.

</details>

<details><summary>C. An error about conflicting writes</summary>

✗ A database guards a row that two writers touch at once, and these workers share one piece of state. Python's dictionary has none: three coroutines assigning to one key raise nothing and guard nothing.

</details>

<details><summary>D. The pricing findings only</summary>

✓ Right. Only the pricing findings reach the report call. Legal's write at 2 seconds was replaced by tech's at 4, tech's by pricing's at 7, and no error was raised. The report covers one topic of the three, and another run may keep a different one, depending on which worker is slowest.

Parallel workers split one task, and their results must all be gathered before the last step combines them. Here the three share one key of one dictionary, which keeps a single value per key, so every write replaces the one before it and the worker that finishes last decides what the report sees.

Have each worker return its reply and pass the list from `gather` to the report call, since that list holds every result in the order the calls were passed, or write each worker's findings under a key of its own, such as `state["legal"]`.

</details>

---

###### 6. Checking a reporter's claims with a lead and workers

A newsroom's fact-checking tool reads each draft article. A first lead call picks the claims that need checking, from three to fifteen per article. Your code hands each claim to a worker: an agent loop with a web search tool, which `run_agent` runs to its end before returning the loop's messages. A second lead call then writes the editor's summary. On Tuesday the search service was down, and one worker's log shows every search failing before it replied "Confirmed: the bridge opened in 1932."

```python
def run_worker(claim):
    messages = run_agent(WORKER_PROMPT + claim, tools=[web_search])
    return messages[-1]["content"]

def fact_check(article):
    claims = call_model(messages=[{"role": "user", "content": LEAD_PICK + article}])
    verdicts = [run_worker(claim) for claim in claims.splitlines()]
    return call_model(messages=[{"role": "user", "content": LEAD_SUMMARY + "\n".join(verdicts)}])
```

When the lead writes the summary, what does it have on the bridge claim?

<details><summary>A. The worker's verdict line alone</summary>

✓ Right. The summary call reads its instructions and the verdict lines, and for the bridge that line is "Confirmed: the bridge opened in 1932.", with no sign in the request that every search behind it failed, so the date can be listed among the confirmed claims.

With an orchestrator and workers, the lead decides at run time how to split the task and combines the workers' results, and here a result is whatever `run_worker` returns: the text of the worker's last message and nothing else. The searches, their errors and any pages a worker read stay in the list `run_agent` returned, which `run_worker` lets go as it returns. The summary is a fresh call whose request holds only what your code put in it, so it can weigh a verdict by its wording alone. A lead built as one agent loop that calls its workers as tools keeps the article in its own messages, yet it too gets from each worker only what that worker returns.

Have each worker return its sources along with its verdict, and let your code mark a verdict as unchecked when none of its searches succeeded, before the lead sees it.

</details>

<details><summary>B. The verdict line and the failed searches behind it</summary>

✗ The lead hands out the work, as a manager does, and a manager can look over each worker's shoulder. The searches and their errors lived in the worker's own messages, and `run_worker` hands back only the last reply. The summary call reads `LEAD_SUMMARY` and the verdict lines, and nothing of how any verdict was reached.

</details>

<details><summary>C. The verdict line with a warning that it was never checked</summary>

✗ The failed searches sit right there in your logs, and logged errors are what usually raise a warning. The log stays with you, and the summary call never reads it. This code compares no worker's searches with its verdict, so the bridge line reaches the lead as the worker wrote it, starting with "Confirmed".

</details>

<details><summary>D. The verdict line and a search the lead runs itself</summary>

✗ A desk chief checks a doubtful date before passing it on, and the lead holds that place here, above workers who can search. The summary call is sent no tools, only `LEAD_SUMMARY` and the verdicts, so its one reply is text, and no search runs between the worker's verdict and the summary.

</details>

---

###### 7. Adding a check pass to a maths study app

A study app for adults going back to college solves maths word problems with one model call. You add a check pass: a second call to the same model gets the problem and the first answer, is told to look for mistakes, and gives the final answer. No answer key or calculator is available to it. You will compare the two versions on problems whose answers you know.

What should you expect the check pass to do to the share of right answers?

<details><summary>A. The share goes up</summary>

✗ Proofreading your own work catches slips, and a second read by the model costs one call. For reasoning, a 2023 study found the opposite: when nothing outside the model checked the work, every model it tested scored lower after reviewing its own answers. On maths problems, one model's changes turned right answers wrong more often than they fixed wrong ones, and this app's second call has no key or calculator either.

</details>

<details><summary>B. The share stays the same</summary>

✗ A model asked to check its own answer has already given the answer it thinks is right. Asked again, it changes some: in a 2023 study one model rewrote about one maths answer in four, and those changes cost more right answers than they won. The app's check pass, reading only the problem and the first answer, has no more to go on, so expect the share to fall.

</details>

<details><summary>C. The share goes down</summary>

✓ Right. Expect fewer right answers with the check pass than without it, at twice the calls per problem. A 2023 study of this kind of pass, a model reviewing its own reasoning with nothing from outside to check it against, found accuracy lower after the review for every model it tried, all of them models of that time, and every test set, maths word problems among them.

A check pass helps only when the review brings in something the first call lacked. Here the reviewer is the same model reading its own work, with no key and no tool, so nothing new enters the second call. In the study, one model kept its first answer to a maths problem about three times in four, and among the answers it changed, it turned a right one wrong more often than a wrong one right. The same study traced the gains reported for such checks elsewhere to the use of the known right answers.

Give the check pass real outside feedback, such as a calculator's result or a test the answer must pass, and measure the pass on your own problems before you ship it.

</details>

---

###### 8. Product pages written by one call and scored by another

A furniture retailer has its product pages written by two calls: a writer drafts each page from the product's data, and a reviewer scores each draft out of 10. Its rubric, a scoring guide, gives no page a 9 unless the page states a warranty term. One sofa arrived from the supplier with no warranty in its data. In this evening's run its four reviews so far scored it 7, 8, 8 and 8, and every one asked for the warranty.

```python
draft = call_model(messages=[{"role": "user", "content": product_brief}])
while True:
    review = call_model(messages=[{"role": "user", "content": RUBRIC + draft}])
    if score_of(review) >= 9:
        break
    draft = call_model(messages=[{"role": "user", "content": product_brief + draft + review}])
publish(draft)
```

What in this code can end this loop for the sofa?

<details><summary>A. Scores that stop rising</summary>

✗ Stopping once a score stops improving is a habit from training models, and here the score has stalled at 8 three times in a row. This loop compares each score with 9 and nothing else. Another 8 means another rewrite and another review, and nothing counts how many 8s came before.

</details>

<details><summary>B. Requests that outgrow the context window</summary>

✗ An agent that appends every step to its history runs out of context window in the end. This loop appends nothing: each review request holds the rubric and the newest draft, and each rewrite the brief, the newest draft and the newest review, so the requests stay about the same size from round to round.

</details>

<details><summary>C. Nothing: the rewrites and reviews go on</summary>

✗ Four reviews in a row asked for a warranty the data does not hold, and no honest draft can supply one. The writer is not held to the data, though. A rewrite that states a warranty gives the reviewer the term it asked for, and the reviewer reads the rubric and the draft, never the sofa's data. A 9 for that draft ends the loop and publishes it.

</details>

<details><summary>D. A 9 for an invented warranty</summary>

✓ Right. The loop ends only on a review that scores 9 or more, whatever the draft says to earn it. A draft that writes in a warranty of its own invention can earn that 9, since the review request holds the rubric and the draft and none of the sofa's data, and the sofa then goes live promising cover the supplier never offered. If the 8s keep coming instead, the job runs until someone stops it.

In an evaluator loop the writer revises and the reviewer judges, and the loop stops only where your code stops it: here the single test `if score_of(review) >= 9`. Nothing compares one score with the last, the requests carry only the latest draft and review, so they never build up toward the context window, and no counter runs anywhere. The reviewer judges the text it is sent, and the sofa's data is not in that text, so it cannot tell a warranty copied from the data from one the writer made up.

Send the reviewer the product data along with the draft, tell it that a fact missing from the data stays missing, and set a step cap: count the rounds, stop at a number you choose, and hand the best-scored draft and its last review to a person.

</details>

---

###### 9. Voting three runs on sarcastic reviews

Guest reviews at a hotel chain are sorted into praise and complaints by a model. In ten test runs, plain reviews were labelled right almost every time, while each sarcastic review was labelled right in roughly two of the ten, with no pattern in which two. You now label every review three times and keep whichever label two or more of the runs agree on.

On the sarcastic reviews, how often does the vote land on the right label?

<details><summary>A. More often than one run, though well below plain reviews</summary>

✗ A 2022 study introduced voting over several sampled answers and found clear gains on maths and common-sense benchmarks. A majority helps where most runs are right. On a sarcastic review each run is right only one time in five, so a majority usually forms around the wrong label, and the vote is right about 10 times in 100.

</details>

<details><summary>B. Less often than one run</summary>

✓ Right. On the sarcastic reviews the vote picks the right label about 10 times in 100, half the 20 of a single run, so on this kind of review the vote added to cut mistakes makes more of them.

Voting keeps the answer that most of the runs return, so it sharpens what most runs already do. A vote of three is right when two or three runs are: with the runs independent, as the lack of a pattern suggests, and each run right 2 times in 10 that comes to 10.4 times in 100, and with each run right 95 times in 100, to more than 99. The tests showed no pattern in which runs miss a given sarcastic review, and every run misses most of them, so the mistakes gather on one kind of review rather than on one run. So the vote fixes the slips the model makes now and then on plain reviews, and on sarcastic ones it settles on the wrong label more often than a single run does.

Score the vote separately on each kind of review, and route the kinds most runs get wrong to a person, or to a different prompt or model.

</details>

<details><summary>C. About as often as one run</summary>

✗ A vote is a kind of average of the runs, and averaging three runs that are each right 20 times in 100 gives 20 again. The vote is right only when two runs are right together, and since each run is right one time in five, that pairing is rarer than a single right run.

</details>

<details><summary>D. Almost as often as on plain reviews</summary>

✗ A panel of three outvotes one member's slip, and on plain reviews it does: the vote comes out right more than 99 times in 100. A panel helps only while each member is right more often than wrong. On sarcastic reviews a run is right once in five, so the two wrong runs usually carry the vote.

</details>

---

###### 10. Capping a helpdesk agent at twelve steps

A university's IT helpdesk runs an agent that answers staff tickets with tools that check a mailbox and read the server logs. `call_model` returns a reply whose `message` goes into the history and whose `calls` lists the tools the model asked for, if any; `run_tool` runs one of those calls and returns its output as text. On a ticket about a mailbox that will not sync, the model is still asking for tools on the twelfth step.

```python
def answer_ticket(ticket):
    messages = [{"role": "user", "content": ticket}]
    for step in range(12):
        reply = call_model(messages=messages, tools=TOOLS)
        messages.append(reply.message)
        if not reply.calls:
            break
        for call in reply.calls:
            messages.append({"role": "tool", "call_id": call.id, "content": run_tool(call)})
    return messages[-1]["content"]
```

What does the member of staff get back?

<details><summary>A. The model's best answer from what it found so far</summary>

✗ A person stopped at a deadline hands in what they have. The model is never told about the cap. It answers only on a reply with no tool call, and all twelve of its replies asked for a tool, so the list holds no answer to return.

</details>

<details><summary>B. An error saying the step cap was reached</summary>

✗ A limit in code is often a guard that raises when crossed, the way runaway recursion stops with an error. This cap is a `for` loop over `range(12)`: when the range runs out, the loop ends quietly and the line after it runs. Nothing raises, and the function returns whatever `messages[-1]` holds.

</details>

<details><summary>C. The note the model wrote with its last tool request</summary>

✗ Every step appends the model's reply, and `messages[-1]` is the newest item. In a step that asks for tools, the results are appended after the reply. On the twelfth step the reply asked for a tool, so the last item in the list is a tool's result, not the model's note.

</details>

<details><summary>D. The output of the last tool that ran</summary>

✓ Right. The staff member gets the raw output of the twelfth step's last tool call, such as a line from the mail server's log, returned as if it were the answer, and the ticket shows as answered with no sign that the run hit the cap.

In an agent loop the run ends when the model decides it is done: a reply with no tool call is the answer, and every other reply asks for more. A step cap stops the loop between steps, and what the run hands back is whatever your code does next. Here that is the last item of the list, and after a step that called tools the last item is a tool result. The model never wrote an answer on this run, so there is none to return.

After the loop, look at the last reply: if it still asks for tools, the run stopped at the cap without an answer, so return a message that says so and put the ticket in a person's queue, with the steps the run took attached.

</details>

---

###### 11. Posting a notice before waiting for approval

Refunds at a food delivery company go through a workflow of steps: load the order, review it, pay it. After each step finishes, your code writes a checkpoint to a database, and to resume a run it loads the latest checkpoint and carries on from there. The review step below announces the refund in the team channel, then waits for a manager: `ask_manager` returns the manager's answer if there is one; if not, it stops the run. The manager approves an hour later, and your code resumes the run.

```python
def review_step(state):
    post_notice(f"Refund for order {state['order']} awaits approval")
    approved = ask_manager(state["run_id"])
    return {**state, "approved": approved}
```

Once the run is resumed, how many notices does the team channel show for this refund?

<details><summary>A. Two, one before the pause and another after the resume</summary>

✓ Right. Two identical notices sit in the channel: the first went out before the pause, the second when the run resumed, so the channel shows two refunds waiting where there is one. The refund itself is paid only once, by the step after the review, but a step that charged a card in place of the post would charge it twice.

Here a checkpoint is written only when a step finishes. `review_step` never finished at the pause, so the latest checkpoint is the one from the step before it, and resuming from it calls `review_step` again from its first line. This time `ask_manager` returns the approval, and everything above it, the notice included, has run twice. The checkpoint holds what the steps returned, not how far into a function the code had got.

Put the post in a step of its own before the review, so the checkpoint written after it records that the notice went out; a side effect that has to stay in the same step must be safe to repeat.

</details>

<details><summary>B. One, posted before the pause</summary>

✗ `ask_manager` sits in the code where `input()` would, a call that blocks until the answer comes and hands it to the next line. The paused function is not kept waiting anywhere. The run stopped, and a resume starts from the latest checkpoint, written after the step before `review_step`, so `review_step` runs again from its first line and posts a second notice.

</details>

<details><summary>C. One, and the resume fails with an error</summary>

✗ Some workflow engines refuse to run a step that has already finished. Your runner has no such guard: it loads the latest checkpoint and calls the next step, `review_step`, which posts again, gets the approval from `ask_manager` and returns without an error.

</details>

<details><summary>D. Two, but only if the process restarted while waiting</summary>

✗ The process stays up, and a running process keeps its functions in memory. The stopped run is held nowhere in memory: resuming always goes through the latest checkpoint in the database, whether or not the process restarted, and that checkpoint comes from before `review_step`.

</details>

---

###### 12. Drafting a goodwill voucher before a supervisor approves it

An airline answers complaints with a workflow, and your code saves a checkpoint to a database whenever a step finishes; a resume loads the latest one and carries on from there. In the step below, `ask_supervisor` returns the supervisor's answer if there is one; if not, it shows the draft to a supervisor and stops the run at that line. The next step sends `state["draft"]` to the passenger if it was approved. A supervisor approves a draft offering a 50-euro voucher, and your code resumes the run.

```python
def draft_step(state):
    draft = call_model(messages=[{"role": "system", "content": VOUCHER_RULES},
                                 {"role": "user", "content": state["complaint"]}])
    approved = ask_supervisor(state["run_id"], draft)
    return {**state, "draft": draft, "approved": approved}
```

Which reply reaches the passenger?

<details><summary>A. The draft the supervisor approved</summary>

✗ The supervisor read a draft and said yes, and approvals are given to the text a person read. Here the approval is tied to the run. The checkpoint the run resumes from was written before `draft_step` and holds no draft, so the step starts over from the top, the model writes a new draft, and `ask_supervisor` returns the yes it already has.

</details>

<details><summary>B. Two replies: the approved draft and a second reply</summary>

✗ Code before a pause runs again on resume, and a repeated post or charge leaves two copies behind. Here the code before the pause is a model call, and a model call posts nothing. Its new draft replaces the old one in `state`, and the step that sends replies runs once, with the new draft.

</details>

<details><summary>C. A draft written after the approval</summary>

✓ Right. The passenger gets the draft the model wrote when the run resumed, which the supervisor was never shown, and the yes given to the first draft lets it through, so a new draft with another amount than the approved 50 euros, or other promises, goes out with an approval on record beside it.

An approval pauses a run until a person answers, and here a checkpoint is written only when a step finishes. `draft_step` stopped partway, so the resume loads the checkpoint from before it and runs the step again from the top, model call included. That call is a new request, and the model's reply can differ from one call to the next even on the same input. Moving side effects into steps of their own does not cover this: the call posts nothing, but running it again can change what goes out.

Write the draft in a step of its own, so the checkpoint holds it, show the supervisor that stored draft, and send exactly the text that was approved.

</details>

<details><summary>D. No reply until the supervisor approves again</summary>

✗ The supervisor approved one draft, and the draft that goes out on resume is a different one. `ask_supervisor` already holds the supervisor's answer for this run, so on resume it returns it at once, and the new draft goes out under the old approval.

</details>

---

## Further reading

- [Large Language Models Cannot Self-Correct Reasoning Yet](https://arxiv.org/abs/2310.01798), 2023: models asked to review and fix their own reasoning with no outside feedback, and how all the models tested scored lower afterwards on every benchmark tried.
- [Self-Consistency Improves Chain of Thought Reasoning in Language Models](https://arxiv.org/abs/2203.11171), 2022: sampling several reasoning paths and keeping the answer most of them reach, and the gains it brought on maths and common-sense benchmarks.
- [ReAct](https://arxiv.org/abs/2210.03629), 2022: the agent loop in which a model alternates reasoning with actions such as searching, tested on question answering and interactive tasks.

Prev: [Memory](../memory/README.md) · Next: [Security](../security/README.md)
