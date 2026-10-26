# Tool calling

Needs: [Context window](../context-window/README.md) · [Structured outputs](../structured-outputs/README.md) · Reviewed October 2026

Tool calling lets a model ask for an action it cannot take itself. Along with the messages, the request describes the functions your code offers, and instead of text the model's reply can hold a request to run one of them, with the values to pass. Whatever code receives the reply runs it and sends the outcome back on the next request.

- **Token**: a piece of text, often a word or part of one; requests are counted, limited and billed in tokens.
- **Context window**: the most tokens a model can handle in one call, counting both what it reads and what it writes.
- **Tool**: a function your code offers the model, such as a stock lookup or a refund.
- **Tool definition**: a tool's name, a plain-language description of what it does and when to use it, and a JSON Schema for its arguments, which lists each argument with its type and says which ones are required; definitions go in the request as data, beside the messages.
- **Loading on demand**: a way to keep most of a large tool set out of the request and let the model search for the tool it needs, so that tool's definition is added only then; some provider APIs offer it.
- **Chat template**: the format that turns the messages into the one text the model reads, marking each message's role, such as user or assistant; with tools, it writes their definitions into that text as well.
- **Tool call**: the part of a reply that asks for a tool: the tool's name and the arguments, and on a provider's API an id for this one call.
- **Tool-call parser**: on your own server, the part that finds tool calls in the text the model writes and returns them as calls. Some servers take it as a start-up setting; others work out the call format from the model's chat template.
- **Schema mode for tools**, often called strict mode: a setting on a tool definition that holds each call's arguments to its schema while the model writes them. Without it, a call's arguments can break the schema, such as a number sent as a string or a required argument left out.
- **Tool result**: a message your code adds after running a tool, carrying the tool's output and the call's id, so each result can be matched to its call; on your own server, some server programs give calls no id and match a result by the tool's name instead. Some APIs take each result as a message of its own with the role `tool`; others take it as a block, a separate part inside a user message.
- **Step**: one round of the loop: a request goes out, the reply holds tool calls, and your code runs them and adds the reply and the results to the messages for the next request. An agent repeats steps until a reply holds text and no call. A turn is a user's message and everything up to the reply that answers it, however many steps that takes.

Who runs a tool, and who finds the call in the model's text:

- **A provider's API**: your code runs the functions you define; some providers also offer built-in tools of their own, such as web search, and run them on their side.
- **Your own server**: the server writes the definitions into the prompt through the model's chat template and finds the calls in the model's text with its tool-call parser.
- **A ready-made app**: it runs its own tools in its own loop, and a coding assistant can ask you before a tool edits a file or runs a command.

On all three, the model writes each call as text and reads each result as part of its input.

Tool choice is a request setting for whether the model calls a tool, and it takes one of four values:

| Tool choice | What the request tells the model |
| --- | --- |
| Left to the model | answer, call tools or both, as it decides; the default when tools are present |
| Forced to any tool | call a tool, whichever it picks |
| Forced to one named tool | call that tool |
| Turned off | answer without calling a tool |

Tool calling bites in the code around the model: the loop that runs each call and answers it, and the history that loop keeps and trims. A demo with one tool and one question exercises almost none of it.

This topic lets you:

- say what has run when a reply asks for a refund, and what the tool definitions decide and cost on every request
- write the loop that answers each call in a reply, and tell what it does when a tool fails or you swap the model behind your own server
- keep a long agent run going when you trim its history or force its tool calls

###### 1. Letting a support bot refund an order

A shop's support bot can now refund orders. Your code sends each customer message to the model together with the definition of a `refund_order` tool, and shows the customer whatever text the reply holds. A customer asks for $30 back on a broken order, and your log shows a reply holding a call to `refund_order` for that order and $30.

What happens to the customer's $30?

<details><summary>A. The money is refunded before the reply reaches your code</summary>

✗ The reply names the tool, the order and the amount, all the details a refund needs, and a request that complete passes for the refund itself. The model writes text, and the call is text: a tool's name and its arguments. The definition you sent holds no code, so neither the model nor the provider has anything to run, and `refund_order` has not been called.

</details>

<details><summary>B. The money is refunded when your next request goes out</summary>

✗ The second request is where the model hears how things went, which makes it the natural moment to credit with the refund. What it carries is a result your code produced after running the tool itself. Sending the call back with no result runs nothing, since the request has no code in it to run.

</details>

<details><summary>C. The money waits for the customer to approve the refund</summary>

✗ If you have used a coding assistant, you have watched it stop and ask before a tool edits a file or runs a command. That pause is something the app builds into its own loop. A model's reply holds nothing back: it hands the call to whatever code receives it, and the code here only shows text.

</details>

<details><summary>D. The customer gets no refund</summary>

✓ Right. The customer gets no refund. The reply holds a tool call, which is a request: the name `refund_order`, the order and the amount. Your code shows the text around it, if there is any, and nothing calls `refund_order`.

The customer may read "I'll process your refund now" above a call that nothing runs, and the $30 never arrives. A week later the order comes back to a person as a complaint, with a log showing a refund that was asked for and never done.

A tool definition is data: a name, a description and a schema, with no code behind it that the model or the provider could run. The model answers in text, and a tool call is text in a fixed shape that asks your code to act. Your code has to read the call, run the function it names with those arguments and send the output back as a tool result for that call; only then does the model learn what happened and tell the customer. That makes your code the one place a refund runs, and the place for a refund limit or a check by a person.

Some provider APIs also offer built-in tools, such as web search, that run on the provider's side and come back with their results in the same reply; a function you define is never one of them. Your own server parses the call out of the model's text, hands it back the same way and leaves running it to the code that sent the request.

</details>

---

###### 2. Adding a returns tool beside the order lookup

In a bookshop's help chat, the model looks up orders with a `get_order` tool described as "Look up an order by its number." You add `track_return`, which reads the returns database, and describe it as "Look up a return by its order number." Since then, questions such as "where is my refund?" go to `get_order` about as often as to `track_return`.

Which change gives the model a reason to pick `track_return` for questions about refunds?

<details><summary>A. Temperature 0 on every request</summary>

✗ A split near half and half is what a coin toss would give, and temperature 0 is the setting you reach for to take chance out of a model's answers. At temperature 0 the likelier tool wins every time, but which tool is likelier for a refund question still rests on the two descriptions, and neither says that refunds belong to `track_return`.

</details>

<details><summary>B. Descriptions that say when to use each tool</summary>

✓ Right. Descriptions that say which questions each tool answers are the one change that gives the model a reason to choose. Once `get_order` says it covers an order's items, payment and delivery, and `track_return` says it covers where a return and its refund stand, a question about a refund fits one tool and not the other.

Until then, about half the customers who ask where their refund is get the delivery status of the parcel they sent back, and the chat's weekly report counts their questions as order lookups.

The model knows a tool only by its tool definition in the request: the name, the description and the schema for the arguments. Your function's code and the database it reads stay on your side. Both descriptions say what each tool looks up by an order number, and neither says which questions it answers, so a refund question fits both and the model splits them. Schema mode changes how arguments are written, not which tool fits, and a good result reaches the model only after it has picked the tool. Write each description to say what the tool does, when to use it and when to use the other one, and give tools names that cannot be confused.

A provider's API turns the definitions into text the model reads, and your own server writes them into the prompt with the model's chat template, so the descriptions decide on both.

</details>

<details><summary>C. Schema mode on both tools</summary>

✗ Schema mode is documented as the way to make tool calls reliable, which makes it the first switch to reach for. It holds each call's arguments to its tool's schema while the model writes them. Which tool fits a question is settled from the descriptions, and those still leave a refund question between the two.

</details>

<details><summary>D. More useful output from <code>track_return</code> each time it is called</summary>

✗ A new colleague learns who handles returns by asking around and remembering what worked. The model remembers nothing from one call to the next. A good result from `track_return` reaches it only after it has picked that tool, inside that one chat, and the next customer's question starts again from the same two descriptions.

</details>

---

###### 3. Connecting a calendar integration with 40 tools

A scheduling assistant used to have three tools. You connect a calendar integration and send all 40 of its tool definitions in full with every request, with no loading on demand. Each definition, with its description and schema, comes to about 150 tokens. A typical chat runs ten requests with short messages, and the model calls a tool on one or two of them.

On the tenth request of a chat, which of the 40 definitions add to that request's input tokens?

<details><summary>A. All 40 definitions</summary>

✓ Right. All 40 definitions count as input on the tenth request, as on the first. At about 150 tokens each, that is about 6,000 input tokens on every request and about 60,000 over a ten-request chat, whether the model calls a tool or not.

The week the integration goes live, input per request jumps for every chat, including chats about nothing but a delivery date. The engineer who reads the usage graph finds no message that grew, and has to trace the jump to a list of tools.

The model reads tool definitions the way it reads messages. Before the request reaches it, they become text it reads in full, and it has to read all of them to know whether any tool fits. Some APIs also add a few hundred tokens of their own instructions for using tools. Send each request only the tools it can use, keep each description as long as it needs to be to tell the tools apart and no longer, and where your system can load definitions on demand, use that for a large tool set.

On a provider's API that text is billed as input and takes room in the context window. Your own server renders the definitions into every prompt with the chat template, and pays for them in memory and time.

</details>

<details><summary>B. No definition adds any input tokens</summary>

✗ The definitions sit in a field of their own, beside settings such as the model's name, and settings cost nothing. The model reads the definitions, though, to choose a tool and write its arguments, and every token it reads is input.

</details>

<details><summary>C. Only the definitions of tools called on that request</summary>

✗ A phone bill lists only the calls you made, and a bill for tools could work the same way. Before any call, the model reads every definition to decide whether any tool fits, so all 40 are input on every request, called or not.

</details>

<details><summary>D. Only the 40 tool names</summary>

✗ A person can pick a tool from a list of names, and the schemas belong with your code. The model needs each description to choose and each schema to write the arguments, and the request carries every definition whole.

</details>

---

###### 4. Swapping the model behind a help desk's own server

A university's IT help desk runs its assistant on its own server, with a `reset_password` tool; your loop runs the calls in each reply and shows the student the first reply that holds none. You swap in a model from a different maker, which writes its calls in a format of its own. The server reads the chat template from the new model's files. Every other start-up setting stays as it was, the tool-call parser included. A student asks for a password reset, and the model answers with a call to `reset_password`.

What does the student see?

<details><summary>A. A message that the password has been reset</summary>

✗ The parser is one line in a start-up command that already worked, and nothing about the swap sent you back to read it. A parser reads one call format, though, and the new model writes another. It finds no call it knows in the reply, so `reset_password` never runs and no result comes back for the model to confirm.

</details>

<details><summary>B. An error from the server, and no reply</summary>

✗ You would expect a server that cannot read what the model wrote to say so, the way a JSON parser raises on bad input. A tool-call parser treats text with no call it recognises as an ordinary reply: it reports no calls and hands the text back. The request succeeds, and nothing in the response marks a problem.

</details>

<details><summary>C. The model's call, shown as plain text</summary>

✓ Right. The student sees the call itself: the tool's name and the student's details, written in the new model's call format, where an answer should be, and the password stays as it was.

The student pastes the strange line into a ticket, and the help desk hears about the swap from students. Chats that need no tool read as normal, so a quick check of greetings and opening hours passes while every request that needs a tool fails the same way.

On your own server the model writes each call as text, and the tool-call parser is what turns that text into a call. A parser reads the format of the models it was written for. Text in another format holds nothing it recognises, so it reports no calls and hands the whole text back as the reply's text. Your loop sees a reply with no calls, treats it as the last reply of the turn and shows it. When you change the model on a server that takes the parser as a start-up setting, change the parser to the one for its format, and keep a test that sends a request needing a tool and fails if no call comes back.

A provider's API reads its own models' calls on its side and returns them as calls, so the mismatch cannot happen there. On your own server it depends on where the parser comes from. A server that works out the call format from the model's chat template follows the swap on its own; a server that takes the parser as a start-up setting, as this one does, keeps reading the old format.

</details>

<details><summary>D. A blank message with the call removed</summary>

✗ When a parser works, the call is lifted out of the text and the reply keeps only what is left, often nothing, and it is easy to assume a parser always takes call-shaped text out. This one lifts out only a call it recognises. The new format holds none, so the parser returns the whole text unchanged and the student reads every character of it.

</details>

---

###### 5. Asking about two mugs at once

A potter sells mugs online, and the store's assistant, which runs on a provider's API, has a `check_stock` tool. A customer asks whether the blue mug and the red mug are in stock. The reply carries two tool calls, `call_1` for the blue mug and `call_2` for the red one. `reply.message` is the assistant message to keep in the messages, and the API takes each result as a `tool` message with the call's id.

```python
reply = call_model(messages=messages, tools=tools)
messages.append(reply.message)

call = reply.calls[0]
result = check_stock(**call.arguments)
messages.append({"role": "tool", "call_id": call.id, "content": result})

reply = call_model(messages=messages, tools=tools)
```

What must the second request carry for the model to answer?

<details><summary>A. Nothing more than it holds now</summary>

✗ The code passed every test that asked about one product, and one result per reply was always enough there. This reply carries two calls, and `call_2` is still waiting for its result, so the model has no count for the red mug to answer with.

</details>

<details><summary>B. A single result giving both counts, in place of the one for <code>call_1</code></summary>

✗ Some APIs want all of a reply's results sent in one message, and a single result for both mugs is a short step from there. Sent together or not, each result answers one call by its id. A single result for both answers `call_1` or `call_2` and leaves the other call without its pair.

</details>

<details><summary>C. The red mug's count, added as text in a user message</summary>

✗ The model reads text either way, and a line saying how many red mugs are left holds the same number a result would. `call_2` waits for a result under its id, and a user message does not give one. Even where the model reads the count, `call_2` stays unanswered in the history of every turn after, and some APIs forbid any message between a call and its result.

</details>

<details><summary>D. A second result, for <code>call_2</code></summary>

✓ Right. The second request needs a result for `call_2` too, under that id. As written it holds the assistant message with both calls and one result, for `call_1`.

Your tests ask about one product at a time and pass. In production the customers who compare two mugs are the ones who get an error or half an answer, and the on-call engineer watches the failures rise and fall with how often people compare.

Parallel calls are several tool calls in one reply, each with its own id, and every one of them needs its tool result in the next request, matched by id. The results may travel together, but each answers exactly one call. Loop over every call in `reply.calls` and append one result per call; if you decide not to run a call, still send a result that says it did not run. Some APIs refuse a request with an unanswered call; where one goes through, the model has no count for the red mug and can only guess or ask again.

The shape differs between APIs, a separate `tool` message for each result or blocks in one user message, and the matching by id is the same in both.

</details>

---

###### 6. A student asking for two receipts

Students of a language school can ask its billing assistant for receipts, and a `send_receipt` tool emails the receipt for one payment. A student asks for the receipts for payments A-1042 and A-1043 and gets three emails: A-1042 twice and A-1043 once. The log shows one reply for that turn, and it holds three `send_receipt` calls, two of them for A-1042. Your loop runs every call in a reply and answers each id.

Which change fixes this?

<details><summary>A. A line in <code>send_receipt</code>'s description saying never to repeat a call</summary>

✗ A description is where you tell the model when to use a tool and when not to, so a rule against repeats has an obvious home there. A description is text the model reads, and nothing checks a call against it. It may make repeats rarer, but a reply can still hold two identical calls, and your loop still runs both.

</details>

<details><summary>B. Running each distinct call once</summary>

✓ Right. The student gets one receipt per payment, and all three calls are answered. Group the calls by tool name and arguments, run each group once and send its output as the result for every id in the group.

At the end of the month the school's bookkeeper finds two receipts for A-1042 in the sent mail and has to check that the payment was taken only once. Around a tool that books a classroom or orders course books, the same loop would book the room twice or order every book twice over.

Parallel calls are separate requests to your code, each told apart by its id, and nothing between the model and your loop merges calls whose arguments match. A repeat is as well formed as the first call, with a name, arguments and an id of its own, so the loop has to expect one, and a check on ids alone never catches it. Make the action safe to repeat as well: send the email service an idempotency key with each email, a value it remembers so it can refuse a second send under the same key. Build the key from the student's message and the payment, so a retry of that one request is caught there, while a request to resend in a later message goes through.

The loop is your code, so the grouping works the same whether the model runs behind a provider's API or on your own server.

</details>

<details><summary>C. Skipping any call whose id has already run</summary>

✗ Skipping an event whose id you have already handled is how a webhook handler survives a delivery that arrives twice, and the habit carries over. Where calls carry ids, each call in a reply has its own, repeats included, so the two calls for A-1042 carry different ids. A check on ids runs all three, and the student still gets A-1042 twice.

</details>

<details><summary>D. Running the calls one after another instead of at once</summary>

✗ Two sends racing each other are the usual picture of a duplicate, and running the calls in order removes the race. Order changes when each send runs, not how many run. One after the other, the two calls for A-1042 each send it, and the student still gets it twice.

</details>

---

###### 7. A weather tool that times out

On a travel app, an assistant answers weather questions with a `get_weather` tool that calls an outside service. On Tuesday the service timed out, and `get_weather` raised `TimeoutError` halfway through a user's turn.

```python
def answer(messages):
    while True:
        reply = call_model(messages=messages, tools=tools)
        messages.append(reply.message)
        if not reply.calls:
            return reply.text
        for call in reply.calls:
            output = TOOLS[call.name](**call.arguments)
            messages.append({"role": "tool", "call_id": call.id, "content": output})
```

What does the user get on that turn?

<details><summary>A. The model's apology that the forecast is unavailable</summary>

✗ A good assistant says when a service is down, and an apology is what a well-built chat gives when a lookup fails. That apology comes after the error was sent back as the call's result. Here the exception leaves `answer` before any result is appended, and the model is not called again on this turn.

</details>

<details><summary>B. A forecast, after the model retries <code>get_weather</code></summary>

✗ The model chose to call `get_weather`, so picturing it noticing the failure and calling again comes naturally. The model never learns the call failed. The exception travels up through your loop, and no request carries a failure the model could react to.

</details>

<details><summary>C. A forecast the model writes from what it already knows</summary>

✗ Models answer plenty of questions from what they learned in training, so a general forecast is a fair guess at a fallback. That fallback would take another call to the model, and the exception ends `answer` before one is made. The user gets nothing from the model at all.

</details>

<details><summary>D. No reply from the model, only the error your app shows</summary>

✓ Right. The user gets no reply from the model. `TimeoutError` leaves `answer` before the call's result is appended, and whatever your app shows for an uncaught error is the whole turn.

The messages now end with an assistant message whose call has no result. When the user asks again, the new message lands after that unanswered call, and every later request in that chat carries a call that was never answered.

A tool result is how a tool's outcome reaches the model, whether the tool worked or failed. Catch the exception around each call and send its text back as the result for that call, such as "Error: TimeoutError: weather service did not answer. Try again in a minute." The model reads it on the next request and can tell the user the forecast is unavailable. Write the error so the model can act on it: what failed and what to try next. The same catch covers a call your function cannot take as written, such as one whose arguments break the schema, which can happen without schema mode. Some APIs have a flag on the result that marks it as an error; others leave the result's format to you.

The loop is your code on a provider's API and on your own server alike, and so is the fix.

</details>

---

###### 8. Pinning the task and trimming the rest of an agent's history

An agent that plans business trips runs on a provider's API. Its messages start with the user's task. After that, each step adds an assistant message with the step's tool calls, usually one call and sometimes two, and then one message for each tool result. Long tasks grow past the size you want to send, so each request will carry the task plus a trimmed part of the history. You are choosing the trimmer, the rule that decides which part that is.

Which trimmer keeps every request in the message format the API documents?

<details><summary>A. The task plus the last 20 other messages</summary>

✗ Pinning the task fixes the trimming bug you have heard about, and with one call per step every step is two messages, so an even count cuts between steps. A step with two calls adds three. Once one sits in the kept part, the steps behind it start an odd number of messages back, and a cut at 20 lands inside a step, keeping a result whose call is gone.

</details>

<details><summary>B. The task plus as many of the newest messages as fit a token budget</summary>

✗ Counting tokens instead of messages is the fix for a slice that ignores size, and it does keep each request under the size you set. The budget, though, runs out at whatever message it reaches, and nothing moves that point to the edge of a step. When it falls between a call and its results, or between two results of one step, the request opens on a result that answers nothing in it.

</details>

<details><summary>C. The task plus as many of the newest steps as fit a token budget</summary>

✓ Right. Every request then holds the task and a run of whole steps: each assistant message with calls comes with every result that answers it, and no result comes without its call.

A trimmer that cuts by count or by tokens passes every test, since a short task never reaches the cut. On the first long trip, a request carries hotel prices for a search that is no longer in it, and the traveller's plan stalls on a history that starts in the middle of a step.

A tool result is half of a pair: it answers the tool call with the same id, and that call sits in the assistant message before it. Provider APIs document the pairing as part of the format: a result follows the call it answers, and every call gets a result. Some refuse a request with a call left unanswered. A trimmer that ignores steps can cut inside one. A count of 20 cuts cleanly while every step holds one call, and slips inside a step once a step with two calls is in the kept part; no fixed count survives every mix of steps. A token budget stops wherever the tokens run out, with no regard for where a step begins. Trim in whole steps: start from the newest step and add older steps one at a time while they fit, so every cut falls before an assistant message with calls and the history after the task begins on a call. When the history must shrink further, replace the oldest steps with one short summary message instead of slicing through them.

Whether an API takes each result as a message of its own or as a block inside a user message, the pairing rule is the same, and a cut between whole steps keeps it on both.

</details>

<details><summary>D. The task, every message with calls, and only the newest results</summary>

✗ Results hold the bulk, a page of search hits against a one-line call, and dropping the old ones saves the most for the least. Every call you keep still waits for its results. Old calls with no results break the pairing as surely as a result without its call does.

</details>

---

###### 9. Keeping a research agent from answering from memory

At a law firm, a research agent sometimes cited cases from memory instead of searching. To stop that, you call your wrapper with `force_tool=True`, which sets tool choice to "forced to any tool". This question assumes that your model, and the API or server it runs behind, both accept and apply a forced choice. Before the change, with tool choice left to the model, runs ended with an answer after a few searches.

```python
def research(question):
    messages = [{"role": "user", "content": question}]
    for step in range(1, 51):
        reply = call_model(messages=messages, tools=tools, force_tool=True)
        messages.append(reply.message)
        if not reply.calls:
            return reply.text
        for call in reply.calls:
            output = TOOLS[call.name](**call.arguments)
            messages.append({"role": "tool", "call_id": call.id, "content": output})
    raise RuntimeError("step limit reached")
```

How does a run end now?

<details><summary>A. At the step cap, with a RuntimeError, on every run</summary>

✓ Right. Every run makes 50 model calls and ends in `RuntimeError`. No reply comes back without a tool call, so `return reply.text` is never reached.

Each question now costs 50 calls, each resending a longer history, and the user waits the longest time your agent allows only to get an error. The research may be done by step 3, and the agent goes on searching to the end anyway.

Tool choice applies to each reply on its own: a forced choice leaves the model no reply without a call, and your loop ends only on a reply without one. Force a tool on the first request only, then leave the choice to the model. Or give the model a `final_answer` tool and end the loop when it calls that one; this needs the choice forced to any tool, since a choice forced to one named tool, such as `search`, makes every reply call that tool. At the step cap you can also send one last request with tool choice turned off, so the run ends with an answer from what it found rather than an error.

A provider's API holds a forced reply to a tool call. Your own server does the same where it supports the setting, by limiting what the model can write, and some servers do not support it at all. Not every model accepts a forced choice: some refuse the request with an error, so check that yours accepts one.

</details>

<details><summary>B. With the model's answer, once it has searched enough</summary>

✗ The change was meant to make the agent search more, and a push it can drop once it has enough would do that job. A forced choice is a rule on every reply this wrapper asks for: each must hold a tool call. Your loop returns only on a reply without one, so it never returns.

</details>

<details><summary>C. With an error from the API, once nothing is left to search</summary>

✗ A model told to call a tool with nothing left to find is in a bind, and you would expect a bind to end in an error. Where a model accepts a forced choice, every reply is held to a tool call. The model calls some tool, however little is left to find, and the API answers normally.

</details>

<details><summary>D. With the model's answer, after its first search</summary>

✗ Some agent libraries reset a forced choice after the first tool call to stop endless loops, and if you have used one, a forced search followed by an answer is what you have seen. Your wrapper resets nothing: each request carries the forced choice, and the API applies it to that request alone. The second reply must hold a call as surely as the first, and so must the fiftieth.

</details>

---

## Further reading

- [Toolformer](https://arxiv.org/abs/2302.04761), 2023: a model that taught itself which tools to call, such as a calculator, a search engine or a calendar, when to call them and what to pass.
- [Gorilla](https://arxiv.org/abs/2305.15334), 2023: a model trained to write calls to real machine-learning APIs, which made up fewer calls than prompted models and, given a document retriever, kept up with changed documentation.
- [The Berkeley Function Calling Leaderboard (BFCL)](https://proceedings.mlr.press/v267/patil25a.html), 2025: a benchmark of the calls models write, single and parallel, of when they hold back, and of multi-step agent tasks, where memory and long-horizon reasoning remain open.

Next: [Cost and latency](../cost-and-latency/README.md)
