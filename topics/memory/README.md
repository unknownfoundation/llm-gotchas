# Memory

Prev: [Retrieval and RAG](../retrieval-and-rag/README.md) · Next: [Agents and workflows](../agents-and-workflows/README.md) · Reviewed October 2026

Memory in an LLM system is text kept outside the model and sent back to it in a later request. Your code or an app decides when that text is written, in whose words, under which key, and which rule brings it back.

- **System prompt**: the instructions meant to hold for the whole chat, sent as the first message of the list or, on some APIs, as a separate field.
- **Short-term memory**: the messages of the conversation in progress, sent with every call while it lasts.
- **Embedding**: a list of numbers an embedding model computes from a text, so that texts with similar meaning get similar lists.
- **Key**: what a memory is filed and looked up under, such as a chat's id, a user's account or a team.
- **Recall**: the rule that picks which memories go into a request, such as all of them, the latest, or the ones a search finds for the new message; it brings back only what was written.
- **Memory tool**: a tool the model can call to read and write memory, such as files in a folder; your code or the app carries out each call.

A provider's API keeps nothing between calls unless you use a feature that stores the conversation on the provider's side, so long-term memory lives in stores your code keeps, and where an API offers a memory tool, your code still holds the files; a model server you run yourself keeps nothing between calls either; a ready-made app keeps your conversations for you, some chat apps also save facts about you and search your past chats in new ones, and a coding assistant reads its memory files at the start of every session.

Memory holds the four kinds of thing in the table below. The conversation is the short-term kind. Facts, episodes and instructions are long-term memory, which papers on agents call semantic, episodic and procedural memory, in that order. In the table, "your code" means your code beside a provider's API or your own server:

| Kind | What it holds | Where you find it |
| --- | --- | --- |
| Conversation | the messages of the chat in progress | your code; some provider APIs, which can store it on their side; every ready-made app |
| Facts | short statements about the user or the world, such as a diet or a home city | your code; some chat apps, which save them as you chat |
| Episodes | what happened in earlier sessions, kept as their messages or as a recap | your code; some ready-made chat apps, which search your past chats |
| Instructions | how to do the work, such as rules, steps and preferences | your code, often in the system prompt; a ready-made coding assistant's memory files |

A **second brain** is long-term memory for a person rather than an assistant: their own notes, cut into short passages and indexed, so that the passages that match a question can be found and sent to a model.

This topic lets you:

- work out what a returning user's next request carries, given where their conversation and facts were kept and under which id
- choose when to write a memory and in whose words, and which rule brings it back when a correction, a booking, a house rule or an old remark has to reach a later session
- give an agent a memory folder, or search your own notes as a second brain, and predict what reaches the model, after a run cut off at its cap or a deploy as well

###### 1. A recipe bot that promises to remember

A recipe app has a chat that works through a provider's API, with no tools for the model to call. Your code sends the model each chat's messages and keeps them until the user closes the chat. On Monday a user writes that they are vegetarian, and the bot replies: "Got it, I'll remember that for next time." On Tuesday they open a new chat and ask for a quick dinner idea.

What does the model have on Tuesday about the user's diet?

<details><summary>A. Monday's chat, sent again by the provider</summary>

✗ Chat apps list yesterday's conversations down the side, and the provider saw every one of Monday's messages. An API carries a past conversation into a call only when your code sends it, or chains the call to a reply the provider stored. This code does neither: Tuesday's request holds the system prompt, the dinner question and nothing else.

</details>

<details><summary>B. A saved note that the user is vegetarian</summary>

✗ The bot said it would remember, in so many words, and a promise from the thing you are talking to is hard to doubt. The promise was text the model wrote, with no store behind it. The model has no tool here, so nothing it writes can save a note, and your code saved none.

</details>

<details><summary>C. No mention of the diet at all</summary>

✓ Right. Tuesday's request carries the system prompt with the dinner question after it, and the word vegetarian appears nowhere in it, so the dinner idea is chosen with no word of the diet.

The model holds nothing over from one call to the next and knows only what a request carries. Monday's messages were short-term memory, and they went when the chat closed; this app has no long-term memory at all. "I'll remember that" is a reply like any other, the words that fit the moment, and a reply cannot write to a store unless your code gives the model a tool for it and carries out the call.

To carry the diet into later chats, write it under the user's id when they state it, and send it with every new chat.

</details>

<details><summary>D. The fact itself, learned by the model on Monday</summary>

✗ The bot answered "I'll remember that", and chat products advertise memory. A model's weights change only in a training run: every call reads the same weights and its own request, so Monday's chat left nothing in the model for Tuesday to draw on.

</details>

---

###### 2. Running four copies of a garden centre's chat

A garden centre runs a chat that calls a provider's API. Your code keeps each conversation's messages in a dictionary inside the running service, keyed by the conversation's id, and sends them with every call. For the spring sale you run four copies of the service, and the load balancer hands each new message to whichever copy is least busy. Most customers write three or more messages in a conversation.

What does the request for a customer's third message carry?

<details><summary>A. The customer's first two messages, whichever copy takes the third</summary>

✗ On a single copy, every message arrives with the ones before it, and the conversation's id travels with each message wherever it goes. The id finds a conversation only in the dictionary that holds it, and each copy keeps a dictionary of its own, filled by the messages that copy happened to take.

</details>

<details><summary>B. The first two for some customers, one or none for others</summary>

✓ Right. The third message's request carries whichever of the first two messages its copy happened to handle; when it lands on a copy that took neither, the request holds the new message alone, and the customer can be asked again for what they already said.

A memory store lives where it is kept, and this one is a dictionary inside each running copy, filled by the messages that copy received. The key, the conversation's id, finds a conversation only in the dictionary that holds it, and a single copy never shows the gap. If the balancer chose a copy at random, each message would land on the copy that took the one before it about one time in four, and on one of the other three the rest of the time. A chat-memory class from an agent library that keeps the history inside the process behaves like this dictionary.

Move the conversations into a store every copy reads, such as a database, and look each one up by its id on every message; pinning each conversation to one copy hides the problem until that copy restarts. Where a provider's API stores the conversation on its side, keep the id of the stored reply in the shared store, since every copy needs it.

</details>

<details><summary>C. The third message alone, whichever copy takes it</summary>

✗ A service built to scale is often made to forget everything between requests, and this one runs as four copies behind a balancer. These copies keep their dictionaries for as long as they run, so a third message that lands on a copy that took the first or the second carries what that copy holds of the conversation to the model.

</details>

---

###### 3. Reopening a Japan trip chat in July

A travel agency offers customers a planning chat. The provider's API behind it stores each reply on its side, and your code sends each new message with only the id of the reply before it. Your code keeps those ids and nothing else of the chats. In April a customer spends an evening planning a trip to Japan; in July they open the same chat and ask to swap the hotel in Kyoto.

What does the model get of the April chat with the July message?

<details><summary>A. No word of the April evening</summary>

✓ Right. The July request names an April reply, and the provider has since deleted it at the end of the period it keeps stored replies for, so nothing said in April can reach the model.

Short-term memory here was kept for you, in a memory store on the provider's side, and that store keeps a stored reply only for a period the provider sets: a day on some plans, several weeks on others, after which the reply is deleted and nothing takes its place. Your code held only the ids, and by July they point at replies that no longer exist.

Keep the messages in your own store as well, under the chat's id, and send the history yourself once a stored reply has expired; or close chats older than the period and open the next one with a recap the customer can see.

</details>

<details><summary>B. Every turn of the April evening</summary>

✗ The chat still opens under the same id, and until now the id was all a new message needed to bring the whole conversation along. How long a stored reply survives is the provider's setting, and on the plans documented it runs from one day to a few weeks. By July every reply from that evening is past the period and deleted, and the id names nothing.

</details>

<details><summary>C. A summary of the April evening</summary>

✗ Long conversations get shortened rather than dropped, which is what compaction does, and April's chat was a long one. Compaction shortens a conversation while it is in use and outgrowing the window. When stored replies reach the end of their period, the provider deletes them and keeps no summary of what they held.

</details>

<details><summary>D. Only the final reply from April</summary>

✗ Each new message names a single id, the last reply's, and one id points at one stored item. Each stored reply links back to the one before it, so the chain behind that id held the whole evening, and every reply in it, the last one included, has passed the provider's period by July.

</details>

---

###### 4. Correcting a coding agent's choice of tests

Your team built its own coding agent on a provider's API, and each task runs as a new session, a conversation of its own. Several times the agent has run the whole test suite, which takes 40 minutes, when the team wants only the tests beside the files it changed. Each time someone corrected it in the chat, and it ran the right tests for the rest of that session.

Which change stops the agent running the whole suite on the team's later tasks?

<details><summary>A. Tell the agent at the end of each session to remember the rule</summary>

✗ The agent followed the correction for the rest of that session, as a colleague would after being told once. Asked to remember, it replies "I'll keep that in mind", or at most writes a note into some file with the tools it edits code with. Your code builds the next session's request from the task and the instructions, and neither the reply nor a stray note goes into it.

</details>

<details><summary>B. Save each session and add the past one whose text best matches each new task</summary>

✗ Past sessions hold the correction word for word, and a searchable store of past sessions is a common way to give an agent memory. A search brings a session back when its text resembles the new task. A session about a date parser shares little with a task about an invoice export, so the correction stays in the store on the day it is needed.

</details>

<details><summary>C. Keep one session open for all of the team's tasks</summary>

✗ The agent ran the right tests for as long as the session lasted. The conversation grows with every task, and each call pays for all of it; once it outgrows the window, the call is refused, or trimming or compaction drops the oldest turns, and the correction about which tests to run goes with them.

</details>

<details><summary>D. Add the rule to the agent's instructions</summary>

✓ Right. The instructions go out with the first request of every session, so each session starts with the rule about which tests to run before the agent has touched a file.

A correction in the chat is short-term memory, and it lasts only as long as that conversation. The rule is an instruction, how to do the work, and a rule that must hold on every task needs a recall rule that brings it every time: the instructions sent with each session. A store of past sessions holds the correction too, as an episode, but a search brings an episode back only when its text resembles the new task.

Write the rule once, in plain words, and keep the instructions short, since they are sent and billed with every session.

</details>

---

###### 5. Pressing New chat in a tax-advice app

An accounting firm gives its clients a tax-advice chat built on the code below, and the app's New chat button calls `new_chat`. `call_model` is your thin wrapper around the model call: it takes the messages and returns the reply text. A client spends the morning asking about rental income, presses New chat after lunch, and asks how much they can pay into a pension this year.

```python
def send(user, text):
    history = store.get(user.id, [])
    history.append({"role": "user", "content": text})
    reply = call_model(messages=[SYSTEM] + history)
    history.append({"role": "assistant", "content": reply})
    store.put(user.id, history)
    return reply

def new_chat(user):
    screen.clear()
```

What does the request for the pension question carry?

<details><summary>A. The system prompt, the morning's messages and the pension question</summary>

✓ Right. The request carries the system prompt, every message of the morning about rental income, and then the pension question, all as one conversation, and each chat opened after lunch adds to it and costs more on each call.

Short-term memory here is filed under a key, and the key is `user.id`, the client's account. Whatever shares the key shares the memory: `new_chat` clears the screen and leaves the store alone, so the next `send` reads back the whole morning under the same id. The request is a list of messages with roles, and it has no field for where one chat ended and the next began.

Give each chat an id of its own, created by `new_chat`, and key the conversation by it. Facts that should carry from chat to chat, such as the client's tax year, belong in long-term memory keyed by the account, and go into each chat on purpose.

</details>

<details><summary>B. The system prompt and the new question alone</summary>

✗ The screen comes up empty under a button labelled New chat, and that is everything the client sees. `new_chat` clears the screen and touches nothing else. The conversation lives in the store under `user.id`, and the next `send` reads every message of the morning back from there.

</details>

<details><summary>C. The system prompt, a summary of the morning, then the pension question</summary>

✗ Chat apps with a memory feature carry what they learned in earlier chats into new ones, often as a summary. Nothing in this code writes one: `send` reads the list stored under the client's id and sends every message in it as it was saved.

</details>

<details><summary>D. The morning's messages marked as an earlier chat, then the question</summary>

✗ The button says New chat, the function is called `new_chat`, and the client sees a boundary between two chats. The request is one list of messages with roles and text, the morning's turns first and the pension question last, with no field for where one chat ended, so the rental questions arrive as earlier turns of the pension conversation.

</details>

---

###### 6. A meal planner open on a phone and a laptop

A meal-planning app stores the facts of each user as one list in a database and sends the list with every request. Each chat works with the list through the functions below. Your code calls `remember` for each new fact a user mentions, and `save_facts` replaces the stored list with the one it is given. One evening a user has a chat open on their phone and another on their laptop. On the phone they mention a peanut allergy, on the laptop that their new home has no oven, and they close the phone chat a minute before the laptop chat.

```python
def open_chat(user_id):
    return {"user": user_id, "facts": db.load_facts(user_id), "messages": []}

def remember(chat, fact):
    chat["facts"].append(fact)

def close_chat(chat):
    db.save_facts(chat["user"], chat["facts"])
```

Which of the two new facts does the user's next chat get?

<details><summary>A. The peanut allergy and the missing oven</summary>

✗ Each chat called `remember` for its fact, and on a single device a fact saved in one chat is always there in the next. Here the laptop's chat loaded the list before the phone saved the allergy, and its save at closing put that older list, plus the oven, in place of the phone's.

</details>

<details><summary>B. The peanut allergy, with the oven lost</summary>

✗ Version control refuses a push that would overwrite a colleague's change, and a database can refuse a write that would replace newer data. `save_facts` stores the list it is given with no check of what changed, and the later save, from the laptop, is the one that stays.

</details>

<details><summary>C. The missing oven, with the allergy lost</summary>

✓ Right. The next chat gets the old facts and the missing oven. The peanut allergy, which the phone saved a minute earlier, is gone from the database.

Each chat took its own copy of the list when it opened, and the laptop's copy was taken before the phone saved the allergy to the database. Writing here happens when a chat closes, and it writes the whole list: the phone's save put the allergy in, and the laptop's save, a minute later, replaced that list with its own copy plus the oven. Two chats of one user overwrite each other as two people editing one record would.

Write each fact when it is said, as a row added on its own, so that no save carries an old copy of the rest; or reload the stored list and merge before saving.

</details>

<details><summary>D. The old list, with neither new fact</summary>

✗ Two saves of one record within a minute are a collision, the case locks and transactions exist for. Each save ran on its own and replaced the list: the phone's stood for a minute, then the laptop's took its place with the oven in it.

</details>

---

###### 7. A guest's remark kept word for word

A hotel chain has a booking assistant that saves each remark a guest makes about themselves, word for word, as a row holding the text and the date it was saved. The code below puts a guest's remarks into every new chat. In March a guest wrote: "I'm in Lisbon for work this week, book me something near the office." In November the same guest asks for a quiet hotel for a weekend away.

```python
def system_prompt(guest_id):
    remarks = store.remarks_for(guest_id)
    lines = "\n".join(remark["text"] for remark in remarks)
    return SYSTEM + "\nAbout this guest:\n" + lines
```

What does the November request tell the model about the guest and Lisbon?

<details><summary>A. That they are in Lisbon for work this week</summary>

✓ Right. The remark goes in as it was typed, in the present tense with "this week" included and no date beside it, though the remark is eight months old, so the guest can be offered hotels near an office they visited in spring.

Writing chose the words, and here they are the guest's own, kept with their time: "this week" meant a week in March and still says this week in every request that carries it. The row's date stayed in the database, since `system_prompt` sends only each remark's text, and the model reads "this week" against the request it sits in.

When you write a memory, turn relative times into dates ("on a work trip in Lisbon, 3 to 7 March"), send each memory's date along with it, and let facts that hold only for a while expire.

</details>

<details><summary>B. That they spent a week in Lisbon in March</summary>

✗ The row holds the date it was saved, so the store knows the remark is from March. `system_prompt` sends `remark["text"]` and nothing else, so the date stays in the database and the request holds the sentence as it was typed, with its "this week".

</details>

<details><summary>C. That they mentioned a work trip to Lisbon at some earlier time</summary>

✗ Eight months on, the trip is long since over, and the store knows when the remark was saved. The request carries it under "About this guest:" in the present tense, with no date beside it, so the model can take it as true this week.

</details>

<details><summary>D. No remark about Lisbon at all</summary>

✗ A weekend away has nothing to do with a work trip. The code makes no choice: it joins every remark the guest has made and sends them all, the one about Lisbon among them.

</details>

---

###### 8. Keeping the last message of a party-booking chat

A play centre takes bookings for birthday parties through a chat. Your code saves a note for each parent with the functions below, keeps nothing else from a chat, and starts the next chat with that note in its system prompt. A parent wrote "The party is on Saturday 7 June, for 23 children.", the bot replied "Lovely, noted: early June, around twenty children. See you soon!", and the parent closed the chat.

```python
def close_chat(parent_id, messages):
    notes[parent_id] = messages[-1]["content"]

def open_chat(parent_id):
    note = notes.get(parent_id, "No earlier chats.")
    return [{"role": "system", "content": BOOKING + "\nFrom the last chat: " + note}]
```

What does the next chat's system prompt say about the party?

<details><summary>A. Saturday 7 June, for 23 children</summary>

✗ While the chat runs, `messages[-1]` is the parent's latest message each time the model is called. By the time the chat closes, the bot has answered, and the last item of the list is its reply. The parent's sentence with the date and the head count was never stored.

</details>

<details><summary>B. Early June, around twenty children</summary>

✓ Right. The next chat starts with the bot's closing reply in its system prompt, early June and around twenty children, and neither 7 June nor 23 appears anywhere in it, so whatever the next chat plans from the note works from about twenty children.

The words in the note are the model's: when the chat closes, the last item of the list is the bot's reply, so the note holds the booking as the bot restated it. A reply retells what it heard in its own words, and those words can round a number, blur a date or carry a slip, which the note then hands to every later chat as if the parent had said it.

Store what the parent said: their message itself, or fields drawn from it, such as the date and the head count, checked against the message before saving. If you keep the bot's reply as well, label it as the bot's, so the next chat can tell the two apart.

</details>

<details><summary>C. Both the parent's figures and the bot's</summary>

✗ A booking record keeps what was asked and what was confirmed, the way an email thread keeps both sides. `close_chat` stores a single item, the content of the last message, and that is the bot's reply. The parent's message went with the rest of the list when the chat closed.

</details>

<details><summary>D. A summary of the whole chat</summary>

✗ A chat app's memory picks things up from across a whole conversation, and this note is made when the chat closes. No model call runs in `close_chat`: it copies the text of the last message as it stands, so the note is one reply, word for word, and no summary is ever written.

</details>

---

###### 9. Restaurants in the city a customer left

A food delivery app has an assistant that keeps a list of facts for each customer: after each chat a model call picks out new facts, your code adds each one to the list, and every chat sends the whole list. A customer who moved from Leeds to York in October is offered restaurants in Leeds tonight, and the request log shows "Lives in Leeds." and "Lives in York." among their facts.

Which change stops the assistant offering restaurants in Leeds?

<details><summary>A. Tell the model in its instructions to trust the newest fact</summary>

✗ When the model picks the wrong one of two things, an instruction is the first fix to reach for. The request carries both lines as plain statements with no dates, so the only sign of which is newer is their place in the list. The instruction gives the model nothing firmer to go on, and on some evenings it guesses Leeds.

</details>

<details><summary>B. Send only the facts a search finds closest in meaning to each message</summary>

✗ Narrowing a long list is sound practice, and a search by meaning keeps the facts closest to each message. The two facts differ by one city name and score close to each other against a delivery request, so a search can bring back both, either or neither.

</details>

<details><summary>C. Set the temperature to 0</summary>

✗ The answer changes from one evening to the next, and changing answers are what temperature controls. At 0 the pick becomes more repeatable, yet both cities still ride in every request, and the pick it settles on can be Leeds as easily as York.

</details>

<details><summary>D. Replace a fact when its subject comes up again</summary>

✓ Right. With the old fact replaced at the moment "Lives in York." is saved, every later request carries York, and Leeds is no longer in the store to be sent.

Writing a fact here adds a line, and adding free text has no step that sees that "Lives in York." answers the same question as "Lives in Leeds.", so both stay, and recall, which sends every fact, brings both. Two homes in one request leave the model to guess, and a guess can go either way.

Fix it where the facts are written: before saving a new fact, look up stored facts on the same subject and replace or retire them, or keep one field per subject, such as the home city, that a new value overwrites. Save the date with each fact as well, so that a newer one can win when two slip through.

</details>

---

###### 10. Starting each physiotherapy session from the last recap

A physiotherapy app has a coach that starts each session with the recap of the previous one and writes a new recap when a session ends, with the code below; `call_model` sends the messages to the model and returns its reply. On Monday a patient spent an hour with the coach building an exercise plan for their left knee. On Wednesday they opened a short session to ask what time the clinic opens on Saturday. On Friday they open the app again.

```python
def start_session(patient_id):
    recap = recaps.get(patient_id, "No earlier sessions.")
    return [{"role": "system", "content": COACH + "\nLast session: " + recap}]

def end_session(patient_id, messages):
    dialogue = [message for message in messages if message["role"] != "system"]
    ask = {"role": "user", "content": "Recap this session in three lines."}
    recaps[patient_id] = call_model(messages=dialogue + [ask])
```

What does Friday's session start with about the knee plan?

<details><summary>A. Monday's and Wednesday's recaps, the plan in the first</summary>

✗ A recap per session makes a diary, one that grows a page at a time. The store keeps one recap per patient: `recaps[patient_id] = ...` puts Wednesday's in the place Monday's held, and `start_session` reads that single entry.

</details>

<details><summary>B. Wednesday's recap, with Monday's knee plan carried into it</summary>

✗ Wednesday's session began with Monday's recap in its system prompt, so the plan was in front of the model while it wrote. The recap call reads `dialogue`, the session minus its system prompt, which is a short exchange about Saturday's hours, so the new recap has no plan to carry.

</details>

<details><summary>C. Wednesday's recap alone, and no knee plan</summary>

✓ Right. Friday's system prompt holds the coach's instructions and Wednesday's recap, a few lines about the clinic's Saturday hours, and the knee plan appears nowhere in it, so an hour of Monday's work is gone from the coach.

Recall here takes the latest of the stored episodes, which is Wednesday's: `recaps[patient_id] = ...` writes each recap over the one before. Wednesday's session did start with Monday's recap in its system prompt, but the recap call reads `dialogue`, which leaves the system prompt out. The new recap was written from a short exchange about opening hours, and a recap holds only what its call read. Feeding each new recap the previous one would turn it into a running summary, which carries old details forward and loses some of them on each rewrite.

Keep every recap with its date and bring back more than the last one, such as the latest few or the ones a search finds, and keep what must last, such as the plan itself, as facts the coach gets every session.

</details>

<details><summary>D. The plan, read back from Monday's messages</summary>

✗ The coach spent an hour on Monday, every message of it typed into this app. This code keeps no messages: `end_session` saves only the recap and `start_session` reads only the recap, so Monday's messages went with Monday's session.

</details>

---

###### 11. Asking a design studio's assistant about last Tuesday

A small design studio has an assistant in its team chat. After each weekly meeting your code saves the recap with the code below. A question is answered from the recaps the search finds for it; `call_model` wraps the model call and hands back its text. The store holds a year of recaps. On Thursday 16 October 2025 a designer who missed that week's meeting, on Tuesday 14 October, asks: "What did the team agree last Tuesday?"

```python
def save_recap(held_on, recap):
    store.add(date=held_on, text=recap, embedding=embed(recap))

def ask(question):
    hits = store.search(embed(question), k=3)
    context = "\n\n".join(hit.text for hit in hits)
    return call_model(messages=[{"role": "user", "content": context + "\n\n" + question}])
```

Which recaps does the request carry?

<details><summary>A. Only the recap of the meeting on 14 October</summary>

✗ Each recap went in with its date, and a meeting is most easily found by its day. The search compares the question's embedding with each recap's and leaves the date out of it. No line here turns "last Tuesday" into 14 October, so that meeting's recap arrives only if its text happens to rank in the three, and it never arrives alone.

</details>

<details><summary>B. The three most recent recaps</summary>

✗ The question says "last", and "last" means the newest meetings. `store.search` sorts by how close each recap's embedding is to the question's and by nothing else, so a recap from last month and one from last spring compete on wording alone.

</details>

<details><summary>C. No recap, only the question</summary>

✗ A recap seldom spells out its own date, and a keyword search for a date nobody wrote down comes back empty. This search ranks every recap by closeness and returns the top three however weak the best match is, so the request always carries three recaps ahead of the question.

</details>

<details><summary>D. Three recaps closest in meaning</summary>

✓ Right. Three recaps go into the request, the ones whose text sits closest to the question, from any week of the year, with the question after them; their dates stay in the store. A June recap that read "The last review of each month falls on a Tuesday." would share two words with the question and could come first, ahead of the 14 October recap.

Recall here is a search, and a search ranks recaps by how close their text is to the question's text: "last Tuesday" is more text to compare, not a date to filter by. The date saved with each recap sits beside it and is never read, and no line of the code reads today's date, Thursday 16 October.

Turn a time in the question into dates in your code, starting from today's date, filter the store to the recaps held in that range, and only then rank by meaning, or send that range's recaps whole. Put each recap's date into the request as well, so the model can say which meeting it is reading from.

</details>

---

###### 12. Adding a rule to a deploy agent's memory folder

Your team runs a deploy agent through a provider's API, with a memory tool: the model can ask to list the files in a memory folder and to read one, and your code carries out each request and returns the result. After a bad release, a team lead adds a file to that folder that says "Never deploy on Fridays." On Friday morning someone asks the agent to ship a hotfix.

When does the rule reach the model?

<details><summary>A. In the first request, with the instructions</summary>

✗ Coding assistants load their memory files as each session opens, and this folder holds the same kind of file. Here the folder sits behind a tool: the tool's description goes out with each request, and the file's text reaches the model only when the model asks to read it.

</details>

<details><summary>B. When the model asks to read the file</summary>

✓ Right. The rule reaches the model as the result of a read the model asked for. Until then the requests carry the memory tool's description and nothing from the folder, and an agent that goes straight to building the hotfix deploys on a Friday with the rule unread.

A memory tool makes recall the model's choice: it reads memory the way it uses any tool, by asking, and the file comes back to it only as a tool result your code returns. Some APIs add an instruction telling the model to check its memory folder before it starts, which makes an early read more likely without making it certain.

A rule that must hold every time is an instruction, and it belongs in the instructions sent with every request. Better still, put it in code: a deploy tool that refuses on Fridays does not depend on what the model read.

</details>

<details><summary>C. When the agent reaches a deploy step</summary>

✗ Rules in some tools fire when a matching action comes up, the way a file pattern or a hook does, and this rule is about deploys. A memory tool does not watch what the agent is doing: the file is read on the model's request, at a deploy step or at any other moment, or never.

</details>

<details><summary>D. In the first request, as a list of the folder's files</summary>

✗ Listing the folder is the cheap first step, the one an agent takes before it reads anything. A listing is itself a tool call: the model has to ask for it, your code runs it, and the names come back as a result, so the first request carries neither the names nor the rule.

</details>

---

###### 13. Stopping a cataloguing agent at its step cap

A city library catalogues new books with an agent that calls a provider's API. Through its memory tool the model can ask to view, create and edit files in a memory folder, which holds one file, the library's tagging rules. Two book tools, `view_book` and `save_tags`, read a new book's record and save its tags to the catalogue; your code carries out every tool request. The instructions tell the agent to record its progress in memory as it goes, and your loop stops a run after 40 steps. Last night's run on 300 new books stopped at that cap, with the trace below.

```text
step  1   view_book  #1
step  2   save_tags  #1
...       view_book and save_tags for books #2 to #19
step 39   view_book  #20
step 40   save_tags  #20
run stopped: 40 steps reached
```

What does tonight's run find in the memory folder?

<details><summary>A. The rules and a note of the 20 books done</summary>

✗ The instructions asked for progress notes, and the agent follows its other instructions. A memory tool writes nothing unless the model calls it, and each of the 40 steps in the trace asked for a book tool: no create or edit ever reached your code, so no note exists.

</details>

<details><summary>B. The rules and last night's whole run</summary>

✗ A coding assistant keeps a transcript of every session, and this tool is called memory. The memory tool stores only what the model asks it to store. The run's messages lived in your loop's list, and when the loop stopped, nothing copied them into the folder.

</details>

<details><summary>C. An empty folder</summary>

✗ A job stopped at a limit is often treated as a failure and rolled back, as a database does with an unfinished transaction. A memory folder has no transaction to roll back, and stopping the loop between two steps touches no file, so the rules the library put there stay where they were.

</details>

<details><summary>D. The rules file alone, as before</summary>

✓ Right. Tonight's run finds the tagging rules and nothing else: the folder is exactly as it was before last night, with no word of the 20 books, so tonight's run can start again at book #1.

A memory tool makes writing the model's choice, as it makes reading: a file changes only after the model asks for a create or an edit and your code carries it out. An instruction to record progress makes such calls likely, not certain, and some APIs add an instruction of that kind on their own whenever the tool is switched on. The step cap stops your loop between two steps, and nothing runs on the way out unless your code runs it.

Record progress in your code instead: after each `save_tags` result, write the book's number to your own store, or when the loop stops at the cap, write a note of what the run finished, which the model can read through the memory tool on the next run.

</details>

---

###### 14. Redeploying a tutoring agent that keeps notes in a folder

A language school tutors its students with an agent built on a provider's API. The model keeps notes through a memory tool, and each lesson is a new session. Your code carries out the model's memory requests in a `memories` folder next to the service's code, on the disk of the container the service runs in, and the agent keeps a file there for each student with their level and the mistakes they repeat. On Monday evening you deploy a new version, which your platform runs in a new container. On Tuesday a student with twenty lessons behind them logs in.

What is in the memory folder when the agent opens it on Tuesday?

<details><summary>A. Every student's file, as on Monday</summary>

✗ Files on a server's disk survive a restart, and the memory tool comes from the provider. The provider holds no part of the folder: your code carried out every create and edit, on a disk that belonged to the old container. The new container was built from the repository, and the repository never held a student's file.

</details>

<details><summary>B. Only the files the model had read, returned by the provider</summary>

✗ Each file the model read travelled to the provider inside a request, so a copy of it passed through the provider's hands. The memory tool's files are held by your code and nowhere else, and opening the folder on Tuesday returns what your code finds on the new container's disk. A new session carries none of the old requests, so no earlier read comes back with it.

</details>

<details><summary>C. No student files at all</summary>

✓ Right. The agent finds no student files: the folder on the new container is empty, or missing until your code creates it, and every note from the twenty lessons went with the old container; the same happens on every deploy that replaces the container.

When memory sits behind a tool, your code picks that place: the provider passes on the model's requests to view, create and edit, and your code decides which disk or database they touch. Data written to a container's own disk goes when that container goes, and a deploy that replaces it starts the new one from the image, the package of your code that every container is built from, which holds nothing the agent wrote.

Point the memory folder at storage that outlives containers, such as a database or a shared volume, a disk that every container mounts, and file each student's notes under the student's id, so that two containers running at once read the same file.

</details>

---

###### 15. Following a link in a consultant's second brain

A consultant keeps a second brain: years of notes, which the code below splits into paragraphs and indexes. An assistant answers each question from the five paragraphs that match it best; `call_model` passes the messages to the model and returns the reply text. Many notes link to others by title, as in `[[Q3 budget]]`, and the notes app opens the linked note on a click. The consultant asks: "What figures did we agree on the 12 May call?" The top match is a paragraph of the note "Client call, 12 May" that reads "Agreed the figures in [[Q3 budget]]." The budget note, a list of costs, has no paragraph among the five.

```python
def index_notes(notes):
    for note in notes:
        for number, paragraph in enumerate(note.text.split("\n\n")):
            store.add(id=f"{note.title}#{number}", text=paragraph, embedding=embed(paragraph))

def ask(question):
    hits = store.search(embed(question), k=5)
    context = "\n\n".join(hit.text for hit in hits)
    return call_model(messages=[{"role": "user", "content": context + "\n\n" + question}])
```

What does the request carry for the link in that paragraph?

<details><summary>A. The words "[[Q3 budget]]" and nothing more</summary>

✓ Right. The link reaches the model as the characters `[[Q3 budget]]` inside the paragraph, and none of the budget note's text comes with it, so the assistant can say that figures were agreed and not which ones.

A second brain is long-term memory for a person, and its recall is a search: it returns paragraphs by how well their own text matches the question, and a link is text like the rest. The notes app follows links when you click; nothing in this code does. The budget note comes along only if one of its own paragraphs ranks in the five, and a list of costs shares little with a question about a call.

Follow links in your code: when a retrieved paragraph links to a note, add that note's paragraphs, or give the model a tool that opens a note by its title. Indexing can also copy the key lines of each linked note into the paragraph that links to it.

</details>

<details><summary>B. The text of the Q3 budget note</summary>

✗ In the notes app one click opens the budget, and links between notes are the point of keeping a second brain at all. The search returns paragraphs and nothing more, and `ask` joins their text as it is; no line of this code opens a linked note, so the link brings none of the budget's figures.

</details>

<details><summary>C. A short preview of the Q3 budget note</summary>

✗ Notes apps show a preview of a linked note when you hover over the link. `index_notes` stores each paragraph's own text and its embedding, nothing else, so the paragraph carries the title in the link and no preview of the note behind it.

</details>

<details><summary>D. Nothing: the paragraph arrives without the link</summary>

✗ On a web page a link is markup, and the reader sees only its label; in the notes app a click opens the linked note. `index_notes` stores the paragraph as written, brackets and all, and `ask` sends that text, so the title in the link reaches the model with the rest.

</details>

---

## Further reading

- [Cognitive Architectures for Language Agents](https://arxiv.org/abs/2309.02427), 2023: a framework that splits an agent's memory into short-term working memory and long-term episodic, semantic and procedural memory, and maps published agents onto it.
- [Generative Agents](https://arxiv.org/abs/2304.03442), 2023: simulated characters that write every experience to a memory stream and bring memories back by recency, importance and relevance together.
- [LongMemEval](https://arxiv.org/abs/2410.10813), 2024: a benchmark of five long-term memory abilities of chat assistants, on which commercial assistants and long-context models showed a 30% accuracy drop across sustained interactions.

Prev: [Retrieval and RAG](../retrieval-and-rag/README.md) · Next: [Agents and workflows](../agents-and-workflows/README.md)
