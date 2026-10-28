# Context window

Prev: [Tokens and tokenizers](../tokens-and-tokenizers/README.md) · Next: [Inside the model](../inside-the-model/README.md) · Reviewed October 2026

A model's context window caps the tokens in one call: the request the model reads and the reply it writes, counted together. It decides how you send a long document and how you keep a long chat going.

- **Token**: a piece of text, often a word or part of one; windows, limits and bills are counted in tokens.
- **System prompt**: the instructions that set how the model should behave, usually sent as the first message, with the role `system`, or as a separate field.
- **Message list**: the conversation so far, as a list of turns with roles: `user` for the person, `assistant` for the model; tool results go in this list too.
- **Chaining**: on some provider APIs, the server stores each reply, and a follow-up call sends only the new message and the stored reply's id.
- **Prompt caching**: the system keeps its work on an opening part of the request that repeats from call to call, so a later call that starts the same way is processed faster and, on a provider's API, billed at a lower rate.
- **Generation cap**: the request's limit on how many tokens the reply may run to, often called max tokens.
- **Output limit**: the most tokens one reply from a model can run to, set for each model by its provider.

On a provider's API, windows on large models run from about a hundred thousand to over a million tokens, and every token you send is billed. On your own server the window is a setting whose default can be far smaller than the model supports, and a long request costs memory and time instead of money. A ready-made app manages the window for you and can drop or summarise old turns.

When one document is larger than the window, something has to be left out, split up or condensed before it is sent. A long chat that outgrows the window has five usual ways to stay inside:

| Way | What it does | Where you find it |
| --- | --- | --- |
| Trimming | drops the oldest messages before each call | your code, on a provider's API or your own server; built in as an option on some of both; a ready-made app can do it for you |
| Summarising | replaces the older messages with a short summary the model writes | your code, on a provider's API or your own server |
| Compacting | shrinks the older turns into a shorter version, made by the API or the app instead of your code | an option on some provider APIs; a ready-made app can do it for you |
| Retrieving | keeps older turns outside the request and adds back only the ones related to the new message | your code, on a provider's API or your own server |
| Moving to a larger window | sends the chat to a model, or a setting, whose window holds more | a provider's API offers models with larger windows; your own server takes a higher context length, its setting for the window size, up to what the model supports |

This topic lets you:

- send a long contract, manual or book, cached or not, and say what comes back and where in the prompt its key pages should go
- explain what a chat bot or an agent read, and what you paid for, after someone edited its saved chat, after a long run of tool calls, or when each reply is chained to a stored one
- keep a chat going for hours, and choose a way of staying inside the window knowing what each costs and what each keeps

###### 1. Correcting the bot's reply in a saved chat

A clinic's booking chat keeps each conversation in a database, and your code builds every request from it. The bot told a patient their appointment is on Monday, but the doctor moved it to Thursday. Before the patient writes again, someone at the front desk opens the saved chat and changes the bot's reply to say Thursday.

What happens on the patient's next message?

<details><summary>A. The model works from the Monday reply it wrote</summary>

✗ Between people, what was said stays said, whatever someone writes down later. Each call reads only the list your code sends, that list now says Thursday, and Monday is nowhere in the request.

</details>

<details><summary>B. The model reads the Thursday reply as its own answer</summary>

✓ Right. The request carries the Thursday text as an `assistant` message, shaped like every reply the model really wrote, and the word Monday is nowhere in it. Thursday is the only answer the model has on record.

The model keeps nothing between calls; it rereads the list your code sends every time. No field marks a message as edited, and nothing checks that an `assistant` message came from the model: one your code wrote by hand goes in like any other. So the saved chat is the bot's whole memory, and whoever can edit it decides what the bot remembers saying.

Guard who can write the stored history, and log edits in a separate table, outside the list you send. In this case the fix works and the patient hears one story; a bug that saves the wrong text would put words in the bot's mouth the same way.

</details>

<details><summary>C. The model gets both replies, with Thursday marked as a correction</summary>

✗ Databases often keep an edit history, and an auditor can see both versions there. The request carries only the list as it is now: no field in it marks a message as changed, and the Monday text is not sent.

</details>

<details><summary>D. The API returns an error, and no reply</summary>

✗ APIs check what you send them, and a reply the model wrote is the model's record, not yours. Nothing checks where an `assistant` message came from: the API accepts one your code wrote exactly as it accepts one the model wrote.

</details>

---

###### 2. Summarising a contract that nearly fills the window

At a law firm, an assistant summarises a contract in one call through a provider's API, with default settings. Today's contract fills almost all of the model's window. A full summary runs a few pages, far below the generation cap you set. The call returns without an error, and the app shows the summary to the lawyer.

What does the app show the lawyer?

<details><summary>A. The full summary</summary>

✗ The cap is the only length limit written in your code, and the summary runs far below it. The window is a second limit, and the input shares it: here the contract takes almost all of it, and the reply runs out of room long before the cap.

</details>

<details><summary>B. A summary that stops partway</summary>

✓ Right. The summary stops when the window is full, well short of the cap, and the call returns normally, so the lawyer reads a summary that breaks off partway. The contract goes in whole, and the reply gets only the room it leaves.

The window holds the input and the reply together, so a contract that fills most of it leaves the reply the remainder, whatever the generation cap says. A provider's API either refuses the request up front when the input plus the cap is larger than the window, or runs it and stops the reply where the window ends; this call returned without an error, so it ran. On your own server it depends on the server: one stops the reply and marks it as cut, and some can be set to drop the oldest tokens and keep writing, which loses the start of the contract instead.

Before you show a reply, check the field that says why it ended, and set the cap from the window minus the counted input.

</details>

<details><summary>C. A full summary that covers only the contract's later pages</summary>

✗ Chat apps drop the oldest messages when a conversation outgrows the window, and the contract's first pages are the oldest text in this request. The request is taken as you sent it, contract included; only the reply gives way, and it stops when the window is full, partway through the summary.

</details>

<details><summary>D. An empty reply</summary>

✗ A database transaction that cannot finish leaves no partial write behind. The model does not plan ahead: it writes token by token until something stops it, and here the window stops it partway, so the lawyer gets the summary up to that point.

</details>

---

###### 3. Translating a whole novel in one call

A publisher's tool translates manuscripts through a provider's API, on a model whose window is about a million tokens. A long fantasy novel comes in at 400,000 tokens, and its translation will run about as long, so the novel and the translation together fill most of the window. The editor wants the whole translation back.

Which plan gets the whole translation back?

<details><summary>A. One call, with the generation cap raised to 450,000</summary>

✗ The novel and its translation leave a fifth of the window free, and the cap is the one length setting in your request, so it is the setting you can raise. A reply is also bound by the model's output limit, often about 128,000 tokens on models this large, and the cap can hold a reply below that limit, never lift it past.

</details>

<details><summary>B. One call, with the reply streamed back piece by piece as it is written</summary>

✗ A long reply that dies halfway is often a timeout, and streaming is the usual cure for those. Streaming changes how the reply arrives, a piece at a time, and leaves alone how long it may run: the translation still ends at the output limit, far short of 400,000 tokens.

</details>

<details><summary>C. One call per chapter, with the chapters joined in order</summary>

✓ Right. Each chapter comes back as a reply well inside the model's output limit, and the chapters joined in order make the whole book.

A reply is bound three ways: by the window, which it shares with the input; by the generation cap you set; and by the model's output limit, the most one reply from that model can run to. On large models the output limit is often a small part of the window: a model that reads a million tokens may write about 128,000 in one reply. Some providers allow longer replies in a batch job, where you hand over requests and collect the results later, up to 300,000 tokens per reply on one of them, still short of this book.

Translate chapter by chapter, send a short list of names and terms with each call so the chapters agree, and join the parts in order; a chapter that fails can be sent again on its own. When you size a long job, check the output limit as well as the window.

</details>

<details><summary>D. One call, with an instruction to translate every page in full</summary>

✗ When an answer comes back shorter than you wanted, a firmer prompt is the first fix you reach for. An instruction shapes what the model writes within its limits, and no prompt moves the output limit: however firmly it asks for every page, the reply stops there.

</details>

---

###### 4. Pasting a whole manual into one request

You build an internal help tool for a factory. An engineer pastes a whole equipment manual into it and asks where the reset switch is. Counted in tokens, the request is larger than the model's window, and your code sends it to the model provider's API with default settings.

What comes back?

<details><summary>A. An answer from the start of the manual, with the end cut off</summary>

✗ A form field on a web page cuts what you paste at its limit and moves on, and an API has limits too. This one makes no cut: with default settings it refuses the request.

</details>

<details><summary>B. An answer from the end of the manual, with the start dropped</summary>

✗ If you know the window mostly from chat apps, this is what you have seen: those apps can drop old turns on purpose to keep a long conversation going. A plain API call with default settings drops nothing; where an API offers to drop the oldest part, it is a setting you turn on, and without it the request fails with an error.

</details>

<details><summary>C. An answer that skims the whole manual</summary>

✗ A person handed a document that is too long skims it, and a model with a large window is sold on taking a whole manual in. A provider's API with default settings does not skim: input larger than the window fails the request before the model reads any of it, so there is nothing to skim.

</details>

<details><summary>D. An error, and no answer</summary>

✓ Right. The API refuses the request with an error, HTTP status 400, before the model reads any of it, and nothing is cut to make it fit, so the engineer gets an error instead of an answer.

An API that cut long input on its own would have to guess which part to drop, and it could drop the question at the end or the very page that answers it. A provider's API refuses unless you turn on an option that drops or summarises the oldest part. On your own server it depends on the server and its settings, and some can be set to cut input to fit. A ready-made app can drop or summarise old turns as a conversation grows.

Your code is the one place that sees the whole request, so count its tokens before you send it.

</details>

---

###### 5. Caching the policy wording in a long claims chat

An insurer's claims chat runs on a provider's API with default settings. Every request opens with the full policy wording, which takes about half the model's window. Prompt caching is on for the wording, and the usage report that comes with each reply shows it read from the cache, at the lower rate, on every call. One chat has run long: the wording plus its history now comes to more than the window, though the history alone is well under it.

What happens when the claimant sends the next message?

<details><summary>A. No reply: the request is refused as too long</summary>

✓ Right. The API refuses the request with an error, and the claimant gets no reply, on this message or any later one while the history stays this long. Reading the wording from the cache did not take it out of the request.

The window counts the whole request, cached part included. Prompt caching changes how the provider processes the opening of a request and what it charges for it; the model still reads all of the wording on every call. So a cached wording that takes half the window leaves the history and the reply the other half, as before caching was on.

Count the whole request before you send it, and trim the history to the window minus the wording and the room the reply needs.

</details>

<details><summary>B. A reply that draws on the wording and the whole history</summary>

✗ The usage report puts the cached wording on a line of its own, at a lower price, apart from the rest of the input. It is one part of the input, counted apart for the bill: the model still reads the wording on every call, and with this history the request is longer than the window.

</details>

<details><summary>C. A reply that draws on the history alone</summary>

✗ A web cache evicts what it holds when space runs short, and the policy wording is cached. What the provider keeps is its work on the wording; the wording itself is still part of your request, and with default settings a request that does not fit is refused whole.

</details>

<details><summary>D. A reply, with the cache skipped and the wording billed at the full price</summary>

✗ A cache that cannot help usually falls back quietly: a miss costs the full price, and nothing errors. Whether a request fits is a separate count over every token in it, cached or not, and this request is longer than the window either way.

</details>

---

###### 6. Running a log-reading agent on a hard ticket

A hosting company runs an agent on its customers' tickets, through a provider's API with default settings: the model asks for a tool, your code runs it, and the next request carries the result. The tool it uses most reads a server log and returns about 8,000 tokens. The model's window is 128,000 tokens. In testing, tickets took five or six steps; a hard ticket in production takes about thirty log reads, and their results come to almost twice the window.

What happens on a hard ticket?

<details><summary>A. The run finishes, the same as in testing</summary>

✗ Each step reads one log, and the code adds one result at a time. The request holds every earlier log as well: tool results go into the message list, the list goes out whole on every step, and it passes the window long before the thirtieth read.

</details>

<details><summary>B. The run finishes, at a much larger bill</summary>

✗ Requests are billed by the token, and a hard ticket reads five times as many logs as a test run. The bill climbs with every step, since the list goes out whole each time, but the window caps the request first: step 17 sends 133,200 tokens, past the 128,000 the window takes, and the API refuses it.

</details>

<details><summary>C. The run stops partway, on an error from the API</summary>

✓ Right. Partway through the run a request grows larger than the window, the API refuses it with an error, and the run ends with no answer for the ticket, after sixteen steps already paid for.

Each step's request carries the whole message list: the system prompt, the tool definitions, the ticket, and every call and result so far. Say the opening takes 2,000 tokens and each step adds a 200-token tool request beside its 8,000-token log. Step 6 sends 43,000 tokens, which is why the tests passed. Step 16 sends 125,000, and step 17 sends 133,200, past the window, and is refused like any other request that is too long.

Keep the run small: have the tool return the log lines that matter, replace each old result the model has already used with a short note, keeping a result after every tool request, and count the request's tokens before every step.

</details>

<details><summary>D. The run finishes, with the early steps folded into a summary</summary>

✗ A coding assistant folds old steps into a summary when a session runs long, and your agent runs on the same kind of model. A provider's API at default settings condenses nothing; folding old steps away is work your code does, or an option you turn on where the API offers one.

</details>

---

###### 7. Listing every warning in a handbook twice the size of the window

A shipping company gives its crews an assistant that answers questions from vessel handbooks, and one handbook is twice the size of the model's window. Before an inspection, a safety officer asks for every warning in it, with the page each one is on.

How should the assistant handle the handbook so the officer gets every warning?

<details><summary>A. Send the handbook part by part into one chat, then ask for the list</summary>

✗ Reading a long book chapter by chapter works for people, and a chat is built for exactly that kind of back and forth. But the model keeps nothing between calls: if your code sends the history, every earlier part goes out again and the request grows back past the window, and if it does not, the model sees only the last part.

</details>

<details><summary>B. Have the model summarise the handbook, then ask the summary for the warnings</summary>

✗ Summarising is what you would do with a long report before a meeting, and the summary itself fits. It keeps what the summarising call judged important, and nothing makes it keep every warning with its page, so the officer can get a tidy list that leaves some out.

</details>

<details><summary>C. Search the handbook for pages about warnings and send only those</summary>

✗ Sending only the parts that matter is the right move when a question points at a few pages, such as the procedure for one valve. A list of every warning is a different task: a search finds pages that match its query, and a warning worded another way, or tucked into a maintenance step, can fall outside the match.

</details>

<details><summary>D. Send each part in a call of its own, then merge the lists</summary>

✓ Right. Each call reads one part and lists its warnings with their pages, and a last step merges the lists. Every page is read by some call, so no warning has to survive a search or a summary to reach the list.

A document larger than the window can be handled two ways, and the task decides which. When a question points at a few places, send only the parts that matter, finding them with a search over the handbook cut into sections. When the task needs the whole document, such as a full list, process it in parts, one call each, and combine the results.

Cut the parts at section breaks, or let them overlap by a page, so no warning is split between two calls, and spot-check the merged list, since a call reading one part can still miss an item. The officer can then check each entry against its page.

</details>

---

###### 8. Trimming chat history to the last 50 messages

You are building a support chat for an online shop, and the conversation lives in a list that starts with the system prompt. `call_model` is your thin wrapper around the provider's SDK: it takes the messages and returns the reply text. To keep calls cheap, every call sends only the last 50 messages. The chat runs long, well past 50 messages.

```python
history = [{"role": "system", "content": "You are the support agent of an online shop."}]

def reply(text):
    history.append({"role": "user", "content": text})
    answer = call_model(messages=history[-50:])
    history.append({"role": "assistant", "content": answer})
    return answer
```

Once the list is longer than 50 messages, what does the model get on the next call?

<details><summary>A. The system prompt plus the last 49 messages</summary>

✗ Every example you have seen puts the system prompt at the top of the list, and it has stayed there in every request so far. The slice does not know that: the request carries whatever list you send, the system prompt is item 0 of yours, and the slice starts after it once the list outgrows 50 items.

</details>

<details><summary>B. The last 50 messages, and no system prompt</summary>

✓ Right. The request carries the last 50 items of the list and nothing else, and the system prompt is not among them. The call succeeds, and from then on the model answers with none of your instructions.

**The system prompt has no special place in the list**: it is item 0, and the slice walks past it as soon as the list is longer than 50 items. In this chat that happens on the user's 26th message, when the list holds the system prompt, 25 exchanges of two messages each and the new message, 52 items, and `history[-50:]` keeps the last 50. The request takes the list as it comes, so nothing fails.

<p align="center"><img src="figures/q08-trimming-chat-history.svg" width="100%" alt="One list of 52 items, its middle items not drawn. A bracket marks the first 2 items, the system prompt and a user message, as not in the request. A second bracket marks the last 50, sent to the model. The first 2 items turn coral at the call."></p>

Keep the system prompt outside the history, add it to every call, and trim by tokens rather than by message count. Code that passes the system prompt in its own field cannot lose it to this slice, though the first user turns still go.

</details>

<details><summary>C. The last 50 messages, and the system prompt remembered from earlier calls</summary>

✗ The chat shows one continuous conversation, and your `history` variable keeps growing. There is no memory between calls: each call reads exactly the list you send, and this one carries none of your instructions.

</details>

<details><summary>D. The last 50 messages, with the SDK putting the system prompt back in front</summary>

✗ SDKs do plenty for you, from retries to headers. A thin wrapper passes on the list it is given: the slice leaves item 0 out, and nothing between your code and the model puts it back.

</details>

---

###### 9. Adding a summariser to a language-learning chat

You add a summariser to a language-learning chat app that runs through a provider's API. Once the history passes a set size, the model writes a summary of it, and the summary replaces the history on later calls. Your logs show that many chats end right after the summariser runs: the learner sends one more message after it, and the lesson is over.

What did the summariser do to the input bill of a chat that ends this way?

<details><summary>A. It made the bill lower than with no summariser</summary>

✗ Summarising is the usual advice for a long chat, and the last call shrank. The summary call read the whole history plus the instruction, and the last call saved less than that, so this chat paid more than it would have without a summariser.

</details>

<details><summary>B. It made the bill higher than with no summariser</summary>

✓ Right. The summary call read the whole history plus its instruction, and the one call after it saved less than that, so this chat paid more input than it would have without a summariser.

A summary is a model call like any other, and it pays back only over later calls, each of which sends the short summary instead of the long history. One such call always saves less than the summary call read. Take turns of a 100-token message and a 200-token reply, the summariser runs once the history passes 2,000 tokens, its instruction is 300 tokens and its summary 400. The summary call reads 2,400 tokens, and the next call sends 500 instead of 2,200, saving 1,700. A chat of 8 turns bills 9,900 input tokens with the summariser and 9,200 without one; at 9 turns it bills 1,000 fewer, and at 12 turns 6,100 fewer.

Before you add one, check how many turns chats run after it would fire; the summary's own output is billed on top.

</details>

<details><summary>C. It left the bill the same as with no summariser</summary>

✗ The learner never sees the summary call, and it produces no reply the learner reads. It is a model call like any other, billed for everything it reads, and one shorter call afterwards does not cancel that.

</details>

---

###### 10. Keeping a running summary of a kitchen quote chat

A firm that fits kitchens runs a quote chat that can last for hours, and your code saves every message in the database. Every 20 messages, your code has the model write one new summary, at most 150 words, from the summary so far and those 20 messages. On later calls, the new summary stands in for both. In one chat the customer gave their budget once, early on. By the time they are choosing worktops, the log shows the budget in the first summary and in none of the recent ones.

Which change can put the budget back into the summary that this customer's requests carry?

<details><summary>A. Raise the summary limit to 400 words</summary>

✗ The first suspect for a lost detail is a squeeze, and 150 words for a chat that runs for hours is tight. But the next rewrite reads only the latest summary and the 20 newest messages, and neither mentions the budget. More room lets a summary keep more of what it reads; it cannot hold a figure its input no longer has.

</details>

<details><summary>B. Tell the summariser to keep every amount the customer gives</summary>

✗ The prompt is where you tell the model what matters, and amounts matter in a quote. The instruction works on what the rewrite reads: the latest summary, which already lost the budget, and newer messages that never gave it. It can protect figures still in the summary; the budget has to reach the input first.

</details>

<details><summary>C. Rewrite the summary every 10 messages instead of 20</summary>

✗ Smaller batches give each rewrite less to fit into the limit. Each rewrite still starts from the summary before it, which has no budget, and rewriting more often gives an early detail more rounds in which it can be dropped.

</details>

<details><summary>D. Write a fresh summary from every saved message</summary>

✓ Right. A summary written from the saved messages reads the message in which the customer gave the budget, so the budget can reach the summary again.

A running summary is written each time from the last summary and the newest messages, and that call reads only what your code sends it. Once one rewrite leaves the budget out, every later rewrite reads a summary without it and newer messages that never mention it, so no change to how the next rewrite works can bring it back. The earliest details are the most exposed: by the 120th message, a detail from the first batch has been through six rewrites, each a chance to drop it under a limit that stays fixed while the chat grows.

The saved messages are the one place the budget still exists. Rebuild the summary from them, in parts if the chat no longer fits the window, and keep the saved messages for as long as a chat can run.

</details>

---

###### 11. Turning on compaction for a study-planning chat

A study-planning chat runs on a provider's API with compaction turned on, so once a session reaches a size you set, the API compacts the older turns on its server. Early in a long session, a student listed the eleven chapters on their exam. Hours later, after compaction has run, they ask which chapters they listed.

What can the model count on having from the early part of the session?

<details><summary>A. Only what the compaction chose to keep</summary>

✓ Right. After compaction the early turns reach the model only as the shorter version the API wrote in their place. If that version kept all eleven chapters, the answer can be right; if it kept only that the student named their exam chapters, the model has nothing better to go on.

Compacting replaces the older turns with a shorter version: a written summary, or an encoded item that a person cannot read but the model can. Either way it carries forward what the process judged important. A default summary can drop something a later turn needs, so some APIs let you replace its prompt, and some keep the most recent turns word for word.

Anything that must survive exactly, such as a list the student will ask about again, belongs in a store your code controls and sends with each call. Without that, a revision plan built hours later can leave chapters out with no hint that the list was ever longer.

</details>

<details><summary>B. The list of eleven chapters kept whole, with the rest of the early turns summarised</summary>

✗ A careful note-taker keeps the lists and figures and condenses the chatter, and compaction is built to keep what matters. The shorter version is chosen by a process that cannot know which detail the student will ask about later: the eleven chapters can come through whole, or as one line saying the student named their chapters.

</details>

<details><summary>C. All the early messages unchanged, packed into fewer tokens</summary>

✗ Compaction is a kind of compression in everyday speech, and compressed files give back every byte. Here the API writes a shorter version of the older turns, and what it leaves out is gone: if it kept only that the student named their exam chapters, the eleven titles are no longer in anything the model reads.

</details>

<details><summary>D. Nothing from the early turns, only the newest ones</summary>

✗ Trimming, the kind of shrinking you know best, throws the oldest turns away, and compaction shrinks a chat too. Compaction writes something in place of the older turns, so the model gets a version of the early session, only a much shorter one.

</details>

---

###### 12. Asking a long trip-planning chat what you asked first

On a travel site, the planning chat can run all evening, so your code keeps every turn in a store. Each request carries the system prompt and the last six messages, and your code retrieves five older turns from the store and adds them too. Three hundred messages after asking for flights from Lisbon to Tokyo in April, a traveller writes: "What was the first thing I asked you tonight?"

Which five older turns go into the request with the traveller's question?

<details><summary>A. Five turns whose text is closest to the question</summary>

✓ Right. The five older turns are the ones that score highest against the traveller's question. The score does not look for the first message, so the request for flights to Tokyo comes along only if its text happens to resemble the question.

Retrieving adds back the older turns related to the new message, and related means alike: the search scores each stored turn by how much it resembles the message that came in last. "The first thing I asked" is about a turn's place in the chat, and no turn's text says where it sits. Say the search gives a turn one point for each word it shares with the new message, common words left out: the flights request shares none and ranks last, while any turn that happens to say "first", "asked" or "tonight" outranks it. A search that compares meaning has the same blind spot, since it still compares text.

Store each turn with its position and time, so your code can answer a question about order from the store, and send the facts the chat must keep all evening, such as the first request and the dates, with every call.

</details>

<details><summary>B. The evening's first message and four other turns</summary>

✗ Your code saved the flights request with every other turn, so it is right there in the store. The search does not look for it by place: it picks turns by likeness to the new message, and the flights request comes back only if it ranks among the top five.

</details>

<details><summary>C. The five earliest turns of the evening</summary>

✗ The question says "first", and a person searching the chat would scroll to the top of the evening. The search scores how much each stored turn resembles the new message, and no turn's text says where it sits, so the earliest turns score no higher than any others.

</details>

<details><summary>D. Five turns that all discuss the April flights to Tokyo</summary>

✗ A search that compares meaning is the step up from matching words, and this question is about the first request. It still compares text with text, and "What was the first thing I asked you tonight?" says nothing of flights, Lisbon or April.

</details>

---

###### 13. Removing the trimmer after moving to the largest window

You moved a support chat to the provider's model with the largest window. Since every chat now fits, you also removed the trimmer that dropped the oldest messages. Traffic grew that month as well. A month later, support reports vaguer answers in long chats, and finance asks why the bill has jumped.

What should you expect the move to have done to the bill and the answers?

<details><summary>A. Added nothing to the bill and left the answers as they were</summary>

✗ Traffic grew the same month, a ready reason for the bill, and a model working inside its window is doing what it was built for. Without the trimmer, though, each call carries the whole chat, so the bill grows with every turn, and accuracy was found falling as the input grew, often well short of the window size each model claimed.

</details>

<details><summary>B. Raised the bill and left the answers as they were</summary>

✗ Long-context models are usually shown passing the needle-in-a-haystack test, finding one planted fact in a huge text, and that test checks only a shallow kind of reading. In RULER, a 2024 benchmark of 17 models, almost all scored nearly perfect on it and still lost accuracy on harder tasks as the context grew, which fits the vaguer answers support reports in long chats.

</details>

<details><summary>C. Raised the bill and made long-chat answers vaguer</summary>

✓ Right. Every call now carries the whole chat, so late calls in a long chat cost many times more than early ones, on top of the traffic. And a 2024 benchmark found that almost all the long-context models it tested lose accuracy as the input grows.

A larger window only raises the limit: each call still sends the whole list, and every token in it is billed as input. Say a turn adds a 100-token message and a 200-token reply. Turn 1 sends 100 tokens and turn 50 sends 14,800, and the totals climb far faster than the chat: 50 turns send 372,500 input tokens in all, and 100 turns send 1,495,000. The accuracy finding comes from RULER, a 2024 benchmark of long-context tasks on 17 models of that time.

Keep a trimmer that cuts by tokens, and save the large window for the requests that need it.

</details>

<details><summary>D. Made long-chat answers vaguer and added nothing to the bill</summary>

✗ More traffic means more calls, and the chat screen shows one short message per turn. Without the trimmer the request carries the whole chat every time, and every token is billed: if each turn adds a 100-token message and a 200-token reply, the 50th turn sends 14,800 tokens.

</details>

---

###### 14. Choosing where the leave chapter goes in a long prompt

A company's HR assistant sends the whole employee handbook with every question, and it fits in the window with room to spare. Before the assistant opens to staff, you decide where in the prompt to put the parental-leave chapter, the one employees ask about most.

Where in the prompt should you expect the model to use that chapter least well?

<details><summary>A. At the start</summary>

✗ Text at the start of a long prompt is the furthest back by the time the model answers, and in this prompt everything else comes after the chapter. Lost in the Middle found the start was one of the strong spots: a passage placed first was among the best used, so the leave chapter at the start of the prompt sits in a good place, not the worst.

</details>

<details><summary>B. Nowhere: every place in the prompt is used alike</summary>

✗ A window is advertised as one number, and the handbook fits under it with room to spare. A 2023 study moved one relevant passage around inside inputs that fit easily, and accuracy changed with its position alone, worst in the middle, so where the leave chapter sits matters even with room to spare.

</details>

<details><summary>C. At the end</summary>

✗ The end of a long prompt is the part the model reaches last, after everything else, so the chapter comes when the model has already read the whole handbook. A 2023 study, Lost in the Middle, found no fade at the end: passages placed last were among the best used, so a leave chapter placed last sits in one of the strong spots.

</details>

<details><summary>D. In the middle</summary>

✓ Right. The middle is where the chapter would be used least well. A 2023 study, Lost in the Middle, found that models often used a passage best near the start or the end of a long input and worst in the middle, even models built for long inputs.

Fitting in the window means the model receives every token; it does not mean the model uses them evenly. The study placed one relevant document among many others and watched accuracy fall as it moved toward the middle of the list. Model makers' prompting guides agree that layout matters and give their own advice: one puts long documents first and the question last, another repeats the instructions before and after the documents.

Keep what matters most at the edges of a long prompt, or better, retrieve only the sections that matter instead of sending the whole handbook.

</details>

---

###### 15. Billing a chained contract Q&A

You build a contract Q&A tool on a provider's API that stores each reply on its server. The first call sends the contract with the first question. Each follow-up chains to the stored reply before it and sends only the new question. A user asks three questions.

What does the third call charge you for?

<details><summary>A. The new question alone</summary>

✗ You send only the question, and an API bills the tokens a request sends. The server holds the rest, and the model reads all of it again on every call, so a chained call charges for the whole chain.

</details>

<details><summary>B. The second reply and the new question, with the contract not billed again</summary>

✗ Most metered services bill a byte once, when it arrives. The chain is read again on every call, so the contract and the first exchange are billed on the third call as they were on the second.

</details>

<details><summary>C. The contract and both earlier exchanges, plus the new question</summary>

✓ Right. The third call charges for the whole chain, though you sent only the new question. Say the first call is 2,000 tokens, each later question 500 and each reply 300: the third call is charged for 3,600 input tokens for a 500-token question, and its reply on top as output.

Chaining saves your code from resending the chat, and the server still feeds the stored chain to the model on each call, so you pay for every earlier token in it again, at a lower rate where prompt caching applies. The usage page shows 2,000, 2,800 and 3,600 input tokens for three short questions, 8,400 in all, though your code sent 3,000.

When a chain grows long, start a new one from a short summary.

</details>

---

## Further reading

- [Lost in the Middle](https://arxiv.org/abs/2307.03172), 2023: models often use a passage best at the start or end of a long input and worst in the middle.
- [RULER](https://arxiv.org/abs/2404.06654), 2024: almost all of 17 models that find one planted fact in a long text lose accuracy on harder tasks as the input grows.

Prev: [Tokens and tokenizers](../tokens-and-tokenizers/README.md) · Next: [Inside the model](../inside-the-model/README.md)
