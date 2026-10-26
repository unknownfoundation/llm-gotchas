# Releasing LLM changes

Needs: [Context window](../context-window/README.md) · [Generation parameters](../generation-parameters/README.md) · [Testing and evals](../testing-and-evals/README.md) · Reviewed October 2026

A release swaps one input of an LLM feature: the model, the prompt, a setting, a tool or the code that builds each request. The questions follow one change from its first users to its rollback.

- **System prompt**: the instructions at the start of every request, usually its first message.
- **Prompt store**: where the saved versions of a prompt are kept for your code to fetch while it runs, so a prompt can change without a deploy.
- **Model version**: one fixed release of a model; some model names always mean one version, and others point to whichever version the provider moves them to.
- **Tool call**: the part of a model's reply that asks to run one of the tools described in the request, with arguments the model chose; unless the provider runs that tool itself, your code runs it and sends back the result.
- **Eval set and pass rate**: a collection of cases with known good answers, and the share of them a release passes on one run.

Behind a provider's API, the provider releases and retires model versions on its own schedule, and a request to a retired version fails, so no version can be kept for good. On your own server the model files and settings change only when you change them, so a rollback can bring the old model back too, if the code names its files and you kept them. Shadow runs, canaries and split tests work the same wherever your code builds the requests.

A release reaches users in one of four ways:

| Way | What the new version gets | Who sees its replies |
| --- | --- | --- |
| Shadow run | a copy of each live request, with its replies logged and thrown away | no one |
| Canary | a small share of live traffic, widened step by step while chosen numbers stay within limits | the users on that share |
| Split test | a fixed share of traffic, often half, for long enough to compare a number between the versions | the users on its share |
| Switch-over | all traffic at once, moved by a deploy or a feature flag | everyone |

A split test is often called an A/B test.

This topic lets you:

- undo a bad week on an LLM feature, and say what a rollback, a new prompt, a changed tool or a switched-off flag reaches
- read a canary, a shadow run, a split test or a release gate on the eval score before it decides whether a change goes to everyone
- take a customer's complaint about one reply and work out what your logs and a retry tell you about it

###### 1. Rolling back a theatre's box-office assistant

A theatre's box-office assistant answers questions about shows and seats. Whenever it starts, it loads the latest system prompt saved in a prompt store. It searches an index of listings that a nightly job builds from the theatre's website, and the job's code sits in your repository. The model is named by one fixed version. On Thursday you roll back to Monday's code, and the assistant restarts.

| Day | What changed |
| --- | --- |
| Tuesday | The box-office manager saves a rewritten system prompt to the store |
| Tuesday | The website switches to the spring season's listings |
| Wednesday | You deploy new code, and the assistant restarts |
| Thursday | Customers report wrong curtain times |

Which of the changes in the table are still live after Thursday's rollback?

<details><summary>A. None of them; the assistant runs as it did on Monday</summary>

✗ The trouble started the day after your deploy, and undoing the latest deploy is the move that has rescued most bad weeks you can remember. This rollback swapped the code and touched nothing else. The restart asked the store for the latest prompt and got the manager's rewrite, and the index still holds what the job built from the spring pages.

</details>

<details><summary>B. The spring season's listings, and nothing else</summary>

✗ The manager's rewrite first reached customers on Wednesday, the day of your deploy, so filing it under that release comes naturally. It arrived then because the deploy restarted the assistant, and every start reads the latest version from the store. Thursday's restart read it again, and the rewrite is still the latest.

</details>

<details><summary>C. The rewritten prompt and the spring listings</summary>

✓ Right. The manager's rewrite and the spring listings are both still live, and only Wednesday's code went back to Monday's.

If the wrong curtain times come from the rewrite or the new listings, the rollback has changed nothing customers see. On Friday a couple reach the stalls twenty minutes after curtain-up, holding a reply that gave them the wrong time, a day after the rollback went out.

A rollback puts the previous version of your code back into service, and only what lives inside that code goes back with it. A prompt store exists so a prompt can change without a deploy, which also means no deploy, forward or back, changes it: Monday's code asks for the latest version as it starts, and the latest is the manager's rewrite. It first reached customers with Wednesday's restart, which is why it looked like part of the deploy. The index is the job's output, and output stays as it was written whichever version of the job's code is deployed. Keep a record for each release of everything it ran on: the code, the prompt version, the index and the model version. Have the code take the prompt version and the index's name from that record instead of the latest, and a rollback then restores the whole record. On your own server, add the model files to the record; the old model comes back only if those files were kept.

</details>

<details><summary>D. The rewritten prompt, and nothing else</summary>

✗ The listings job lives in your repository, and rolling the repository back to Monday feels as if it takes the job's work back too. A rollback changes which code runs, and the index is data the job has already written. Monday's version of the job reads the same website tonight and builds the spring listings again.

</details>

---

###### 2. Widening a bank chat's canary after one day

A bank's support chat moves to a new system prompt. For one day, 5% of chats run on it as a canary, and a dashboard compares them with the rest: the error rate and reply times match, and cost per chat comes out 6% lower. The rollout plan widens the canary to half of all chats the next morning.

After the canary's day, how correct are the new prompt's answers compared with the current prompt's?

<details><summary>A. More correct than the current prompt's</summary>

✗ The finance lead reads a 6% saving as the bot getting to the point, with customers helped in fewer messages. It can as easily mean shorter replies or customers giving up. Cost counts tokens, and a right answer and a wrong one of the same length cost the same.

</details>

<details><summary>B. Not known yet</summary>

✓ Right. Not known yet. A wrong answer is an ordinary reply: the call succeeds, arrives on time and costs what its tokens cost, so none of the three numbers can tell a right answer from a wrong one.

Suppose the new prompt has the bot quoting an overdraft fee the bank dropped last year. From tomorrow half of all customers can get it, and the first sign may be a customer at a branch counter holding up a screenshot of the chat.

A canary widens a change step by step while chosen numbers stay within limits, and it can only stop for what those numbers measure. The error rate counts calls that failed, reply time counts seconds, and cost counts tokens; a fluent wrong answer clears all three. The 6% could come from shorter replies, from customers who gave up sooner or from a prompt with fewer tokens, and none of those says whether a reply was right. Check the answers before the canary, with an eval, and give the canary a number that moves when answers go wrong, for instance how often a chat ends with the customer asking for a person. A provider's API and your own server both return a wrong answer as a success, so that number has to come from your side.

</details>

<details><summary>C. As correct as the current prompt's</summary>

✗ For a web service, a release that holds its error rate and response times is a release that works, and a green canary says exactly that. A model's wrong answer is not a failed call. It comes back on time with a success status, so these lines stay level whatever the answers say.

</details>

<details><summary>D. Less correct than the current prompt's</summary>

✗ A prompt that makes the model spend less might be one that makes it try less: you get what you pay for. Spending says nothing about correctness: a shorter reply can be a sharper one, and a long reply can be wrong from its first line.

</details>

---

###### 3. A one-hour canary for a pharmacy chat's dosing fix

A pharmacy's chat sometimes gives a dose where it should send the customer to a pharmacist, in about 1 chat in 2,000 by your log review. A rewritten system prompt is meant to stop it. The team gives it a one-hour canary, about 250 chats, and agrees to promote it if the hour is clean. The hour can end clean, with no dosing reply, or with at least one.

Which ending of the hour can tell you whether the new prompt has stopped the dosing replies?

<details><summary>A. A clean hour</summary>

✗ No dose in 250 chats feels like proof, the way a clean test run does. At 1 in 2,000, the old prompt itself would have given a clean hour 88% of the time, so the hour looks the same whether the new prompt fixed anything or not.

</details>

<details><summary>B. A clean hour and a dosing reply alike</summary>

✗ A check that can pass or fail usually tells you something either way, and both results seem to count, but these two are lopsided. A dose shows the failure is still there; a clean hour is what 250 chats give most of the time even at the old rate, and it tells you almost nothing.

</details>

<details><summary>C. Neither a clean hour nor a dosing reply</summary>

✗ Two hundred and fifty chats against a failure that rare sounds far too few to show anything at all, and for a clean hour that is so. A single dose is different: it can only happen if the new prompt still gives doses, so one is enough to rule out a cure.

</details>

<details><summary>D. A single dosing reply</summary>

✓ Right. Only a dose can: one dosing reply in the hour shows the change has not stopped them. At the old rate of 1 in 2,000, 88% of such hours would have come through without a dose as well.

Promoting on that hour sends the prompt to every customer on a result that would have looked the same had it done nothing. In that case about one customer in 2,000 goes on reading a dose in a chat window instead of being sent to the pharmacist on duty.

A canary sees only as many chats as its share and its hours give it. At 1 in 2,000, the failure is expected 0.125 times in 250 chats, about once in eight such hours, so most hours come up clean whichever prompt runs. Even if the new prompt doubled the failure, 78% of hours would still come up clean. The two results do not weigh the same: a dose rules out a fix that works, while a clean hour fits a full cure, a partial one and none at all. A 95% chance of seeing the old failure at least once takes about 6,000 chats, a full day of this canary, and telling stopped from rarer takes far more. Test a rare failure before the canary: send the eval cases that drew a dose to the new prompt many times each, and count. Keep the canary for faults common enough to show in the chats it gets.

</details>

---

###### 4. Trialling a newer model behind an airline's rebooking agent

An airline's rebooking agent can move a passenger to another flight with a tool, and the booking system emails the passenger about every change to her booking. For a trial of a newer model, `on_message` also hands every message to a pool of background workers that runs the agent on that model, and your code never reads what the pool returns. `call_model` is your thin wrapper around the model call: it takes the model, the messages and the tool definitions, and returns the reply's text and tool calls; `run_tool` runs one tool call. A passenger asks about a delayed flight: the current model's reply says she keeps her seat, and the newer model's reply holds a call to rebook her on the 18:40.

```python
def answer_with(chat, model):
    reply = call_model(model=model, messages=chat.messages, tools=TOOLS)
    for tool_call in reply.tool_calls:
        run_tool(tool_call.name, tool_call.arguments)
    return reply.text

def on_message(chat):
    trial_pool.submit(answer_with, chat, NEWER_MODEL)
    return answer_with(chat, CURRENT_MODEL)
```

What has the passenger received an hour later?

<details><summary>A. A reply saying she keeps her seat, and an email that she is on the 18:40</summary>

✓ Right. Her chat says she keeps her seat on the delayed flight, and her inbox holds the booking system's email that she is now on the 18:40. The pool ignored the newer model's text after its rebooking had gone through.

She queues at the gate of the delayed flight and learns there that she was moved to the 18:40 by the same chat that told her to stay. With a full flight behind her, the gate agent has to pick which of the two answers to honour.

In a shadow run the new version's replies are logged and thrown away, and that is all the throwing away covers. A tool call in a reply is an instruction for your code, and `answer_with` carries out every one before it returns, on the trial's thread as on the live one. By the time the pool's result is ignored, the seat has moved and the booking system has sent its email. A shadow has to be kept from acting: point the trial at a test copy of the booking system, which accepts a rebooking and moves nobody, and route that copy's emails to a mailbox the team reads. The test copy's list of rebookings is then the trial's result, the passengers the newer model would have moved. Whether the model sits behind a provider's API or on your own server, your code is what runs the tools, so the email goes out either way.

</details>

<details><summary>B. A reply saying she keeps her seat, and nothing more</summary>

✗ From where she sits the trial is invisible: one chat, one reply, the current model's. Being invisible in the chat is a different thing from doing nothing. The loop in `answer_with` ran the rebooking on the trial's thread before that thread's text came back to be ignored, and the email went out with it.

</details>

<details><summary>C. A reply saying she keeps her seat, and an email asking her to confirm the 18:40</summary>

✗ Every airline site you have used asks you to confirm a change before it counts, so a rebooking she never asked for should wait for her yes. Nothing waits here: the tool moves her the moment `run_tool` runs it, and the email is the booking system reporting a move already made, with nothing for her to answer.

</details>

<details><summary>D. A move to the 18:40, undone when the trial's reply is thrown away</summary>

✗ Work with databases teaches that unfinished work is rolled back, and a run whose answer nobody uses has the feel of unfinished work. There is no transaction around a tool call here. The rebooking finished inside `answer_with`, and ignoring the text afterwards reverses none of it.

</details>

---

###### 5. Judging a newer model from a shadow log of whole chats

A broadband provider is trialling a newer model for its support chat as a shadow run: each request the chat sends to the current model is also sent to the newer one, and the newer model's replies are written to a log. Customers only ever see the current model's replies. To judge the newer model on long troubleshooting chats, the support lead reads the log one chat at a time.

When the newer model writes its second reply in a chat, which messages is it replying to?

<details><summary>A. The customer's first and second messages, with no reply between them</summary>

✗ The trial is there to write replies of its own, so handing it only what the customer typed sounds like the clean way to copy a chat. The copy is the live request itself, and that list holds the assistant's turns as well: the newer model's second request carries the current model's first reply between the customer's two messages.

</details>

<details><summary>B. Only the customer's second message, with nothing before it</summary>

✗ If you have mirrored web traffic to a test service, each copied request stood alone, and a chat message looks like one more of those. The copy here is of the model request, and in a chat that request carries the conversation so far, since a model remembers nothing from one call to the next. The second request holds the whole first exchange.

</details>

<details><summary>C. Its own first reply, then what the customer said to the current model</summary>

✗ A trial that keeps its own copy of the chat, with its replies where the live ones were, is a natural design to picture. This trial copies the live request, which holds the replies the customer saw and none of the newer model's. Its own first reply appears nowhere in what it is sent.

</details>

<details><summary>D. The current model's first reply, and the customer's answer</summary>

✓ Right. The newer model's second reply answers the current model's first reply and the customer's response to it. Only the first reply in each chat answers an opening the newer model would have met on its own.

The support lead reads twenty turns of the newer model calmly working through a router fault and marks it ready for long chats. Every turn after the first rested on steps the current model chose, so how the newer model steers a fault from its own start is the one thing the log cannot show.

A shadow run sends a copy of each live request, and in a chat a request is the whole list so far: the system prompt, every customer message and every reply the customer saw. Those replies are the current model's, so from the second turn on the newer model is handed a conversation another model has been steering and asked to carry on. If its own first reply would have asked for the router's model number while the current model said to restart the router, the customer's next message is about the restart, and the log records the newer model reacting to that. Read a shadow log of chats as first replies to real openings plus continuations of the current model's chats. To see the newer model hold a conversation from start to finish, give it conversations of its own. A canary on a small share of chats does that, and so does a test in which another model plays the customer, starting from a recorded chat's opening. Wherever the model is served, the copied request carries the replies the customer saw, so the log reads the same way.

</details>

---

###### 6. Splitting an online grocer's help chat between two versions

An online grocer splits its help chat between two versions for two weeks, about 20,000 chats each. The new version has a new system prompt and a "Talk to a person" button under every reply, and both show thumbs buttons. The thumbs-down share is 8.4% of 1,964 votes on the new version and 12.0% of 2,046 on the current one, and fewer customers ask the same question twice on the new version. The customer service director wants it for everyone.

Which number can tell you whether the new version answers better?

<details><summary>A. The thumbs-down share, over a month of votes</summary>

✗ When a number moves and you do not trust it yet, waiting for more data is the sound habit. Noise is not what separates these shares: on these counts they already sit about 3.8 standard errors apart, and a standard error is the size of gap that chance alone usually makes. The gap comes from who votes on each side, and a month of the same votes repeats it with more confidence.

</details>

<details><summary>B. The share of chats with a question asked twice</summary>

✗ A customer who asks the same thing twice was not helped the first time, and that signal needs no button at all. The button reaches it anyway: on the new side an unhappy customer can ask for a person instead of asking again, while the current side offers no such way out, so the two shares count different customers.

</details>

<details><summary>C. Grades on a random sample of each side's chats</summary>

✓ Right. Grades given to a random sample of chats from each side, scored the same way by a person with a rubric or a judge model, are the one number both sides produce alike.

The new version goes to everyone on the strength of the drop, and the operations manager takes one person off the evening chat rota to match the better answers. Every press of the new button now lands in a queue with one person fewer to answer it.

A split test gives both sides the same kind of customers, so any number taken the same way on both sides compares them fairly. User feedback is not taken the same way here. It counts only customers who choose to press a thumb, and the new side hands an unhappy customer a second thing to press, a request for a person, in place of a vote or a repeated question. Picture both versions giving a bad answer in 1 chat in 10. An unhappy customer votes 12.3% of the time on the current side and a third less often on the new one; a happy customer votes 10% of the time on both. Down votes then make up 12.0% and 8.4% of the votes, the dashboard's figures, with answers exactly as good on both sides. Adding the presses to the thumbs-downs, or leaving out chats with a press, swaps one difference between the sides for another. Draw the same number of chats at random from each side and have them graded without the grader knowing which version wrote them.

</details>

<details><summary>D. Thumbs-downs and "Talk to a person" presses added together</summary>

✗ If the button drew unhappy customers away from the thumbs, putting their presses back has the look of a fair repair. A press is not a verdict on the last answer: customers ask for a person to change a delivery slot or report a missing item. And the current side had no button, so its unhappy customers left no presses to add.

</details>

---

###### 7. Assigning a split test message by message in a pet shop's chat

A pet shop is testing a new system prompt for its advice chat against the current one. For each customer message your code asks a split service which prompt to use. `split.assign` returns "current" or "new": the same answer every time for the same key, and half and half across different keys. `new_request_id` gives every call a fresh key. A chat runs to about six replies, and when it ends the customer answers "Did this solve your problem?". At the end of the test the survey holds about 40,000 answers on each side: 80% solved on the new prompt, 79% on the current one. `call_model` is your thin wrapper around the model call: it takes the messages and returns the reply text.

```python
def reply(chat, text):
    chat.version = split.assign("advice-prompt", key=new_request_id())
    chat.messages.append({"role": "user", "content": text})
    system = {"role": "system", "content": PROMPTS[chat.version]}
    answer = call_model(messages=[system] + chat.messages)
    chat.messages.append({"role": "assistant", "content": answer})
    return answer

def record_survey(chat, solved):
    survey.add(version=chat.version, solved=solved)
```

What does the survey's one-point lead for the new prompt measure?

<details><summary>A. Chats on the new prompt solve a point more than chats on the current one</summary>

✗ A fresh draw for every message is as random as a split can get, and randomness is what makes a test fair. It makes the version of each chat's last reply a fair draw, and no more. In chats of six replies, 31 in 32 of the chats on either side heard both prompts, so neither side holds chats that one prompt ran.

</details>

<details><summary>B. Chats whose last reply came from the new prompt solve a point more than the others</summary>

✓ Right. The point compares chats whose last reply came from the new prompt with chats whose last reply came from the current one, and in chats of six replies 31 in 32 of each group heard both prompts.

The shop moves everyone to the new prompt on the strength of the lead. A customer asking about a puppy's diet now gets six replies from one prompt, a chat the test never ran, and the shop has no number for how those chats go.

A split test compares two versions only when each side's chats were run by one of them. This code draws a fresh key for every message, so it splits by request: a chat of six replies stays on one version only when six draws in a row agree, 1 chat in 32. Each reply is written on top of the other version's earlier replies, which the request carries in its history, and `chat.version` keeps only the latest draw, so the survey's verdict on the whole chat goes to whichever prompt spoke last. That draw is random, so the lead is a fair comparison of last replies, and at 40,000 answers a side it sits about 3.5 standard errors from no difference. A standard error is the size of gap that chance alone usually makes. The lead is not a comparison of the prompts over whole chats, which no chat in the test ran. Use the chat's id as the key, so every message in a chat draws the same version, or the customer's id where one customer opens several chats. The split is made in your code, so where the model runs changes nothing here.

</details>

<details><summary>C. Noise that more chats would wash out</summary>

✗ A point is small, and small gaps in a survey often melt away once more answers arrive. With about 40,000 on each side this one is unlikely to: chance alone would rarely produce a gap 3.5 standard errors from none. That the gap is real is settled; what it is a gap between is not.

</details>

<details><summary>D. Chats on the new prompt solve well over a point more than chats on the current one</summary>

✗ Mixing two prompts in one chat blurs a test, and a blurred test is expected to understate the gap it looks for. Here the blur can run either way, since a prompt may do worse after replies it did not write as easily as better. No chat in the survey ran on one prompt, so it gives no size for whole chats on the new one, larger or smaller.

</details>

---

###### 8. Shipping a stricter system prompt for a homework helper

A homework helper saves every chat with a student in a database, and a chat can stay open for weeks. Today's release changes `SYSTEM_PROMPT` so the helper gives hints on graded work instead of full answers. `call_model` is your thin wrapper around the model call: it takes the messages and returns the reply text. Tonight a student who opened a chat last week asks it to solve a graded problem.

```python
def open_chat(student_id):
    chat = {"student": student_id,
            "messages": [{"role": "system", "content": SYSTEM_PROMPT}]}
    db.save(chat)
    return chat

def reply(chat_id, text):
    chat = db.load(chat_id)
    chat["messages"].append({"role": "user", "content": text})
    answer = call_model(messages=chat["messages"])
    chat["messages"].append({"role": "assistant", "content": answer})
    db.save(chat)
    return answer
```

Which system prompt does that student's request carry tonight?

<details><summary>A. Last week's prompt, which allows full answers</summary>

✓ Right. The request carries last week's prompt, saved with the chat the day it opened, ahead of every turn. Today's release reaches only chats opened from now on.

Tonight a student with a chat open since last week gets a worked solution to the graded problem, while a classmate who opened a chat this morning gets hints. The teacher who asked for the change hears that it works for some students and not for others.

The system prompt is only the first message of the list your code sends, and this code saves the list with the chat, so the prompt in force when a chat opened travels with it until the chat ends. The release changed a constant that only `open_chat` reads. A rollback has the same blind spot the other way round: chats opened while a bad prompt was live keep it after the code goes back. A switch-over reaches open chats only if each call builds the system prompt fresh. Keep it out of the saved messages, store its version beside the chat if you want a record, and put the current one in front of the saved turns on every call. Your code builds the list on a provider's API and on your own server alike, so both behave this way.

</details>

<details><summary>B. Today's prompt, which asks for hints on graded work</summary>

✗ The prompt lives in your code, and every other change you deploy reaches every user on their next request. The constant is read in one place, `open_chat`, which runs once per chat. Last week's chat was saved with last week's prompt as its first message, and `reply` sends that saved list.

</details>

<details><summary>C. Both prompts, last week's first</summary>

✗ The code holds one prompt and the saved chat holds another, and you may picture both going out. Only the saved list is sent: `reply` loads it and appends the student's message, and nothing puts `SYSTEM_PROMPT` in front of it again.

</details>

---

###### 9. Approving a queued refund after the refund tool changes

At a car-rental company, the support agent refunds customers through a `refund` tool. A refund over 100 euros waits for a supervisor's approval, and the queue saves the call exactly as the model wrote it. Last night the agent queued a refund of 180 euros for a customer whose car broke down, while the tool's definition still gave `amount` in euros. This morning's release changes the definition and the approval code below. Then the supervisor approves.

```python
REFUND_TOOL = {
    "name": "refund",
    "description": "Refund a rental. amount is in cents.",
    "parameters": {"booking": "string", "amount": "integer"},
}

def approve(queued_id):
    tool_call = refund_queue.load(queued_id)
    args = tool_call["arguments"]
    payments.refund(booking=args["booking"], cents=args["amount"])
```

How much does the customer get back?

<details><summary>A. The 180 euros the agent meant</summary>

✗ The call was written before the release, and a change you ship usually leaves finished work alone. This work was not finished: it was waiting to run, and it ran on today's code. `approve` hands the saved 180 to the payment service as cents, and nothing in the saved call says it was meant in euros.

</details>

<details><summary>B. No refund, and an error on approval</summary>

✗ A tool with a typed definition invites the picture of a check that turns away a call written for the old one. Nothing checks the call when it is approved, and a check would pass it anyway: 180 is a whole number, which is all the definition asks of `amount`. The payment service gets a valid request and pays it.

</details>

<details><summary>C. The 180 paid out as cents</summary>

✓ Right. The payment service refunds 180 cents, 1.80 euros. The saved call says `amount` is 180, written last night against a definition in euros, and this morning's code reads it as cents.

The customer who was stranded on a motorway last night opens an email about a refund of 1.80 euros and calls back angrier than before. Every refund queued before the release and approved after it is short a hundredfold, and the queue gives no sign of which ones those are.

A tool call is the model's reading of a tool's definition at the moment it wrote the call. Saved and run later, the call keeps that reading, and nothing in it records which definition it was written against: the queue holds the name `refund` and the number 180, and this morning both mean something new. A release that changes what a tool's field means reaches every call written after it, and also every call written before it and run after it. Store the definition's version with each saved call, and have the approval code run older calls under the meaning they were written for, or send them back to be written again. A changed meaning is safer under a new name, such as an `amount_cents` field, which an old call does not have. Approval runs in your code and calls no model, so where the model is served makes no difference.

</details>

---

###### 10. Taking a new model out of a university's admissions bot

A university's admissions bot answers applicants' questions about deadlines, fees and documents. Its answers are kept for a week in a store that all the bot's servers share. On Monday morning you emptied the store and switched on the flag `new_model`, which sends the bot's model calls to a new model instead of the current one. The late-applications question was first asked that morning and has been answered hundreds of times since. On Wednesday its answer turns out wrong, and at noon you turn `new_model` off. `call_model` is your thin wrapper around the model call: it takes the model and the messages and returns the reply text.

```python
WEEK = 7 * 24 * 3600

def answer(question):
    key = "faq:" + normalise(question)
    hit = shared_store.get(key)
    if hit is not None:
        return hit
    model = NEW_MODEL if flags.on("new_model") else CURRENT_MODEL
    text = call_model(model=model, messages=[{"role": "user", "content": question}])
    shared_store.set(key, text, ttl=WEEK)
    return text
```

When does the late-applications question first get the current model's answer?

<details><summary>A. On the first ask after noon</summary>

✗ Turning the flag off at noon was meant to send the next applicant to the current model, and for a question nobody has asked yet, it does. The late-applications answer is already in the store, and each ask finds it there and returns before the code gets to the flag.

</details>

<details><summary>B. A minute or two after noon</summary>

✗ Flags you have flipped before took a minute to reach every server, and that lag is real. It never touches this question, whose answer comes out of the shared store; the code reads the flag only when the store has nothing.

</details>

<details><summary>C. Once the bot's servers restart</summary>

✗ You have restarted a service before to make a config change stick, so a restart feels like the step that completes a switch. The store is shared by all the servers and sits outside them. A restart empties none of it, and the first ask afterwards finds the same answer.

</details>

<details><summary>D. Once the stored answer expires</summary>

✓ Right. Not until the stored answer expires, a week after the new model wrote it on Monday morning. Until then every ask of that question is served from the store before the code reaches the flag, and the current model is never called for it.

Applicants who ask on Thursday and Friday still read the wrong deadline, and some post their forms a day late. When the admissions team tries the bot with the question worded afresh, the current model answers it, and the staff begin to wonder whether the applicants misread.

A feature flag acts at the line that reads it, and in `answer` that line runs only when the store has nothing for the question. The key is built from the question alone, so it carries no trace of which model wrote the entry, and the entry the new model wrote on Monday satisfies every later ask until its week runs out. A question first asked after noon does reach the current model, and a restart changes nothing, since the store lives outside the servers. A key that names its author mends this: with the model version and the prompt version beside the question, the flag's move points every ask at a key the store has never filled. When you switch a model off, delete what it wrote as well, or those entries return the day the flag does. Saved summaries and any other stored text a model wrote behave the same: the flag stops new writes and leaves the old ones in place.

</details>

---

###### 11. A release gate that compares with the last release's score

The release gate of a tax-filing assistant passes a release only if its pass rate on the eval set is at least the last release's recorded score, 84%. Since release 21 set that score, the team has added 60 cases drawn from customer complaints to the 300 the set had. Release 22 scores 79% on the 360 cases, and the gate blocks it.

What should the team do next?

<details><summary>A. Hold release 22 back until it scores 84%</summary>

✗ The gate has worked this way for months, and a lower number has always meant a weaker release, but the set changed under it. The 79% is a share of 360 cases, 60 of them drawn from complaints, while the 84% was a share of the old 300, and a release ahead on every old case can still score lower once those cases join.

</details>

<details><summary>B. Run release 21 on the 360 cases</summary>

✓ Right. Score release 21 on the same 360 cases, then compare. The 84% and the 79% are shares of two different sets, and side by side they cannot say whether release 22 is better or worse.

While the comparison stands, the release built to handle the complaint cases waits, two engineers spend the week hunting a regression that may not exist, and customers filing before the deadline keep meeting the answers that drew the complaints.

A pass rate belongs to the eval set it was taken on, so a release gate compares like with like only while both scores come from the same cases. Say release 22 passes 86% of the old 300 cases and 45% of the 60 new ones: it scores 79%, and if release 21 passes 30% of the new cases, release 21 scores 75% on the 360 and loses on both parts. Say instead release 22 passes 80% of the old cases and 75% of the new: it scores 79% again, now worse on the old cases than release 21's 84%. Treat the eval set as versioned: write each score down with the set it was taken on, and when cases are added, rerun the last release before its number is compared with anything. A score belongs to its grader as well: swap the judge model or edit its rubric, and grade the last release again with the new grader before the two scores meet.

</details>

<details><summary>C. Run release 22 again</summary>

✗ At 360 cases a score does move from run to run, which makes a second run a fair instinct. It tells you how much the 79% wobbles; it is still a share of a different set from the 84%, and the two stay apart however many times release 22 runs.

</details>

<details><summary>D. Ship release 22 as it stands</summary>

✗ The complaint cases were added because the bot failed them, so they could carry the whole drop. The scores fit that story and its opposite: release 22 can lead on the old cases and trail on the new, or the reverse, and both give 79%.

</details>

---

###### 12. A backup model that answered every chat during an outage

A food delivery app runs its support chat on a model behind one provider's API and, after a minute of errors, switches to a backup model on a second provider, sending it the same system prompt, tools and messages. The backup was chosen after a trial at release 20. The prompt has changed in each of the eleven releases since, and each passed a gate that runs the eval set on the main model. Yesterday the main provider was down for three hours, and the backup answered every chat.

What should the gate run before release 32 goes out?

<details><summary>A. The eval set on the main model, as before</summary>

✗ Eleven releases have passed this gate on the main model, and that model answers almost every chat. Almost is where the gap sits: for three hours yesterday the backup answered all of them, with release 31's prompt, which no eval had ever sent it. Release 32 would reach the backup untried in the same way.

</details>

<details><summary>B. A test that the switch to the backup still works</summary>

✗ Failover drills check that traffic moves when a provider goes down, and they are worth keeping. A working switch proves only that the backup receives the chats. What it says to customers under release 32's prompt is a separate matter, and only an eval run on the backup can answer it.

</details>

<details><summary>C. The eval set on the backup, after each outage</summary>

✗ Checking the backup once it has been needed sounds thrifty, since it matters only when the main provider fails. By the time you check, its answers have already gone out. An outage gives no notice, so the backup has to be checked with each release's prompt before the release, as the main model is.

</details>

<details><summary>D. The eval set on both models</summary>

✓ Right. The gate has to run release 32's prompt and tools through the backup as well as the main model. The backup answers customers whenever the main provider fails, so it carries each release as surely as the main model does.

For three hours yesterday, customers asking about late orders were answered by the backup reading a prompt eleven releases newer than the one it was tried with. If one of those releases added a refund rule the backup handles badly, the incident report blames the outage, and the refunds have already been promised.

A release is whatever answers customers, and a backup model takes release 32's prompt and tools to a model the gate has never run them on. The main model's pass rate speaks for the main model with that prompt. The trial at release 20 spoke for the backup with release 20's prompt, and each of the eleven releases since changed what the backup would be asked. Put the backup inside the release gate: run the eval set through both models and hold the release back if either misses the bar. Treat a switch to the backup as a switch-over of the model, and send it a small share of chats from time to time, so its replies on the current prompt are read before an outage puts them in front of everyone.

</details>

---

###### 13. Building a claims bot's eval set from its complaints

A home insurer is giving its claims bot a new prompt. Its eval set is the 150 chats customers complained about over the last three months, each with the reply it should have given. Run on the set, the current release passes 21 of them and the new prompt passes 96, and the release notes promise a much better bot.

What does the eval set need before it can back what the release notes promise?

<details><summary>A. More complaint chats, from earlier months</summary>

✗ Complaints are the cases you already know how to collect, each one a failure worth having in the set. More complaints tell you more about the failures and still nothing about the chats that go well, which are most of the bot's traffic.

</details>

<details><summary>B. Each of the 150 chats run five times</summary>

✗ Running every case several times is how you cope with replies that differ from run to run. Repeats steady the score on the 150 complaint cases; all of them are chats the bot once got wrong, so the repeats cannot show what the new prompt broke among the everyday ones.

</details>

<details><summary>C. Everyday chats sampled from traffic</summary>

✓ Right. A random sample of everyday chats from traffic, scored for both releases. Every chat in the set is one the bot once got wrong, so none of the everyday chats it gets right is there to show what the new prompt broke.

The new prompt ships as an improvement. If it now fumbles a routine question it used to get right, such as how to report a burst pipe, every customer who asks that gets the fumble, and the eval still shows 96 of 150.

The eval set and its pass rate speak only for the cases in it, and these are cases the current release once got wrong. A new prompt changes every reply, not only the ones it was written to fix, so a case that passed before can fail after; such a case is often called a negative flip. Take a day of 2,000 chats in which the current release handles 1,900 well: a new prompt that fixes 64 of the 100 bad ones and turns 95 good ones bad handles 1,869 well, 93.5% against 95%, while passing far more of the complaint cases. Keep the complaint cases as a set of their own, add a sample drawn at random from traffic, and compare the two releases case by case on that sample, so every flip from pass to fail shows up by name.

</details>

<details><summary>D. A second reviewer grading the 150 chats</summary>

✗ Two graders often split on the same reply, which makes a second pair of eyes the usual cure. Better grading makes the 96 a firmer score on complaint cases. It adds no case the bot handles well today, so whatever the new prompt broke stays out of sight.

</details>

---

###### 14. Pasting a customer's messages in again after a wrong contract date

A mobile network has an account assistant that searches the network's help pages for each question, and a job rebuilds that index every night. On Monday a customer complains that on Friday the assistant told her that her contract ends in March, when it ends in May. Your log keeps each chat's messages, so you send her messages to the assistant again, and it answers May.

What can you conclude about Friday's March from Monday's May?

<details><summary>A. Its cause is still unknown</summary>

✓ Right. Monday's May says nothing about where Friday's March came from. Monday's request held her messages and the pages Monday's search found; Friday's request was never saved, and the same request sent twice can come back with two different answers.

She has already given notice to move to another network in March, and an early-exit fee arrives on her next bill. The agent on the phone has only Monday's answer to go on and assures her the assistant now gives the right month.

A reply comes from one whole request: the system prompt, the settings, the messages, the pages the search returned at that moment and the model version that answered. The chat log keeps one of those parts. Since Friday the index has been rebuilt every night, so Monday's search can return other pages, and even an identical request may not get the same answer twice. A retry built from the chat log is a new request, and its answer describes Monday. Log each request in full as it goes out, the pages included, with the prompt version and the model version the response reports. With Friday's pages in hand, the first question settles itself: either March was in what the search gave the model, and the pages need fixing, or the pages said May and the model wrote March anyway, and the fix belongs in the prompt or the model. Only with the whole request saved can a change be tried against the case that failed.

</details>

<details><summary>B. Its cause has been fixed</summary>

✗ When a bug stops showing up, someone has usually fixed it, and nothing on your side looks broken any more. Nobody changed anything here on purpose, though. Monday's search ran over an index rebuilt since Friday, and an answer that comes back right once can come back wrong when a request meets Friday's conditions again.

</details>

<details><summary>C. It was a one-time slip</summary>

✗ A model's answers vary from call to call, and an error that does not come back has the air of a fluke. One right answer cannot tell a fluke from a fault that returns one time in five, and Monday's request was not Friday's request to begin with.

</details>

<details><summary>D. Friday's pages were out of date</summary>

✗ The nightly rebuild is the one change since Friday you can name, which makes it the natural suspect. Monday's answer cannot convict it, since nobody saved the pages Friday's search returned, and with those same pages the model could still have answered May on a second try.

</details>

---

## Further reading

- [Positive-Congruent Training](https://arxiv.org/abs/2011.09161), 2020: cases an old model got right and its replacement gets wrong, called negative flips, and a way to train image classifiers that lowers the error rate and the flips together.
- [MUSCLE](https://arxiv.org/abs/2407.09435), 2024: updating the base model under fine-tuned adapters turned some of their right answers wrong, and a training method cut those flips by up to 40% in its tests.
- [Hidden Technical Debt in Machine Learning Systems](https://papers.nips.cc/paper_files/paper/2015/hash/86df7dcfd896fcaf2674f757a2463eba-Abstract.html), 2015: the costs that pile up around a system built on a model, among them entanglement, hidden feedback loops, configuration and changes in the outside world.

Next: [Where models run](../where-models-run/README.md)
