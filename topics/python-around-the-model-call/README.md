# Python around the model call

Prev: [Cost and latency](../cost-and-latency/README.md) · Next: [Streaming](../streaming/README.md) · Reviewed October 2026

The Python around a model call builds the request, sends it, waits, handles failures, and parses, checks and stores the reply. The call crosses a network, takes seconds and returns text a model wrote, so habits from ordinary Python meet delays, failures and inputs they were not built for.

- **Model call**: one request to the model and the reply that comes back. In the code on this page, `call_model(messages=...)` stands for a small function of yours that sends the message list and hands back the reply text; `acall_model` does the same with `async def`.
- **Message list**: the conversation sent with every call, a Python list of dicts, each with a `role` and a `content`. The system prompt, the standing instructions for the whole chat, often goes first with the role `system`.
- **Prompt template**: a string with named slots that your code fills before each call. An f-string is filled at the point in the code where it appears, from the names visible there; `str.format` fills an ordinary string later, from the arguments you pass it.
- **Thread pool**: a fixed set of worker threads that run the functions you hand to it, several at a time.
- **Future**: the object a thread pool gives back for each function you hand it, standing in for that function's result until the result is ready.
- **Async function**: a function defined with `async def`; inside it, `await` marks each place where it waits for something, such as a reply.
- **Coroutine**: the object that carries out an async function's body, able to pause at an `await` and later pick up where it stopped.
- **Event loop**: the part of asyncio that runs coroutines, resuming each one when what it is waiting for is ready; a program usually has one.
- **Task**, in asyncio: a coroutine handed to the event loop with `asyncio.create_task`, so it runs alongside the code that created it.
- **`asyncio.gather`**: takes several coroutines and, when all of them succeed, returns their results as one list, in the order they were passed.
- **Generator**: the object a function containing `yield` returns; it produces the function's values one at a time, as a loop asks for them.
- **Embedding**: a vector, a list of numbers, that a model computes from a piece of text so that a search store can find texts close in meaning.

On a provider's API every call that completes is billed for its tokens, and each maker's client library brings its own error classes and time limit; your own server often takes the same request format, so the same Python and client libraries talk to it, and a call costs the server's time instead of money. The behaviour described here is that of Python 3.11 and later.

The code around a call works in seven steps, and each has Python of its own:

| Step | What your code does | Python you meet there |
| --- | --- | --- |
| Building the prompt | fills templates and puts messages into the list the call sends | f-strings, `str.format`, lists and dicts |
| Sending the call | sends one request or many at once, and waits for the replies | blocking calls, thread pools, `async def`, `await`, `asyncio.gather`, tasks |
| Handling failures | catches what goes wrong, retries or gives up, and reports it | `try` and `except`, timeouts, retry loops |
| Parsing the reply | turns the reply's text into Python values | `json.loads`, a `dataclass` built from the result |
| Checking the reply | tests those values before anything acts on them | `is True`, `__post_init__` and other checks you write |
| Acting on the reply | runs what the reply decides, such as a refund, a tool or a message | `if` tests on the parsed values |
| Storing the results | saves replies, embeddings and records of each call for later | files, `json.dump`, generators, tasks |

This topic lets you:

- build the prompt and the message list in Python and know what text each call carries, for every customer and every chat
- run many calls at once under time limits and retries, and say which calls ran, which were paid for and what the error report will show
- take a reply apart, check it, act on it and store it, and find the line where an odd reply or a second run goes wrong

###### 1. Dinner-time traffic on an async recipe chat

A recipe site runs its cooking chat on an async web server, as one process with one event loop, and each chat message is handled by an `async def` function. Inside that function your code calls the model through the provider's synchronous client, and a reply takes about six seconds. At dinner time, two cooks send a message one second apart.

What happens to the second cook's message while the first reply is on its way?

<details><summary>A. It is picked up at once and answered in about six seconds</summary>

✗ Async servers are chosen to serve many users at once, and the handler is written with `async def`. The loop switches only at an `await`, and the synchronous client holds the one thread for all six seconds; until it lets go, the server cannot even read the second message.

</details>

<details><summary>B. It waits until the first reply is back</summary>

✓ Right. The second message sits unread until the first reply returns, five seconds after it arrived, and only then does its own six-second call begin. The second cook waits about eleven seconds for an answer that takes six, and on a busy evening each message waits for every reply ahead of it.

An async function lets go of the thread only where it stops at an `await` for something not ready, and only there can the event loop move to another coroutine. A blocking call never awaits: the synchronous client holds the thread until its reply arrives, and with the thread held, the loop can neither read the second message nor start its call.

Call the model through an async client and await it, or pass the blocking call to `asyncio.to_thread`, which runs it in a worker thread and leaves the loop free.

</details>

<details><summary>C. It is turned away with an error from the server</summary>

✗ Web servers under load answer with a busy error, and this one is busy for five seconds. This server refuses nothing: the second message waits, unread, until the event loop gets its thread back, and then it is handled like any other, five seconds late.

</details>

<details><summary>D. Both replies come late, each slowed by the other</summary>

✗ An operating system shares one processor between programs by giving each a slice of time, so two jobs at once both run slower. The event loop does not slice time: it changes coroutines only at an `await`, so the first call keeps the thread until its reply is back, and the second starts after it, at full speed and late.

</details>

---

###### 2. Summarising book descriptions overnight with a thread pool

An online bookshop has its 950 book descriptions summarised overnight. Your script hands each description to a pool of eight worker threads with `submit`, keeps none of the futures it gets back, and once the pool has shut down, logs how many descriptions it submitted. One description is a whole sample chapter pasted in by mistake, and the model call for it raises an exception.

What does the morning log show?

<details><summary>A. A traceback, and a run that stopped at the failed description</summary>

✗ In a plain script, an exception that nobody catches stops the program, and this call raised one. It was raised in a worker thread, where the pool caught it and kept it in that call's future; the main thread never asks for the result, so it never hears of the failure and carries on to the end.

</details>

<details><summary>B. A finished run, with the exception's traceback printed partway through</summary>

✗ A thread you start yourself prints a traceback when an exception kills it. A pool's worker is not killed: it stores the exception in the future and moves on to the next description. Nothing is printed unless your code asks that future for its result.

</details>

<details><summary>C. No end to the run: the pool hangs on the failed thread</summary>

✗ A lock that is never released freezes a whole program, and a failed worker never reports back. The failure sits in that call's future and never reaches the pool's shutdown, which finishes on time with every thread free.

</details>

<details><summary>D. A finished run, and no word of the failure</summary>

✓ Right. The night's log shows 950 descriptions submitted and nothing more: no traceback, no failure. Of the 950 descriptions, 949 get a summary, and the book with the sample chapter is the one without.

A thread pool runs each function you submit on a worker thread and records the outcome in that submission's future: the value it returned, or the exception that came out of it. The exception waits there until your code asks the future for its result, and then it is raised again in your thread. Here no one asks, so nothing is raised, printed or logged, and the futures are thrown away with the exception inside.

Keep the futures and call `result()` on each, or loop over `as_completed`, and count the failures as well as the successes.

</details>

---

###### 3. Two actions in one triage reply

A helpdesk lets a model triage tickets. Your prompt asks for a JSON object with an `action` field, and your code parses the reply with `json.loads` and runs the action it names. For ticket 5180 the model writes an object that holds `action` twice: first `"archive"`, then `"delete"`.

What happens to ticket 5180?

<details><summary>A. <code>json.loads</code> raises, and the ticket stays as it was</summary>

✗ Some JSON parsers reject an object with a repeated key, and a reply that names two actions is exactly the input you want refused. Python's `json` module accepts repeated names: it keeps the value that comes last, silently, and your code runs that action.

</details>

<details><summary>B. The ticket is archived</summary>

✗ Archive is the first word the model wrote, and a person reading from the top would act on it before reaching the second. A dict keeps one value per key, and as `json.loads` reads the object, each repeat replaces the value before it. By the time the object closes, `action` holds `"delete"`.

</details>

<details><summary>C. The ticket is deleted</summary>

✓ Right. The ticket is deleted. The dict that `json.loads` hands back is `{'action': 'delete'}`, with no trace of the `archive` the model wrote first.

Parsing the reply turns its text into one Python dict, and a dict holds one value per key. The JSON standard asks for unique names in an object but leaves open what a reader does with repeats; Python's `json` module, like many readers, keeps the last value. A model can write the same key twice, and the parser does not object.

To catch a repeat, pass `json.loads` an `object_pairs_hook` that raises on a name it has already seen, and check a destructive action before it runs.

</details>

<details><summary>D. The reply parses, and the ticket stays as it was</summary>

✗ Libraries that read query strings or form data often keep every value of a repeated name as a list, and a list matches no action. `json.loads` never builds one: the second value overwrites the first, `action` is the plain string `"delete"`, and your code deletes the ticket.

</details>

---

###### 4. Showing the reply's shape inside an invoice prompt

An accounting firm extracts fields from scanned invoices. Your prompt template is a plain string with an `{invoice_text}` slot, and to show the model the shape you want, it ends with a one-line JSON example.

```python
PROMPT = """Extract the vendor and the total from this invoice.

{invoice_text}

Reply in this form: {"vendor": "Vendor Name", "total": 0.00}"""

def extract(invoice_text):
    prompt = PROMPT.format(invoice_text=invoice_text)
    return call_model(messages=[{"role": "user", "content": prompt}])
```

What happens when the run reaches the first invoice?

<details><summary>A. No request at all: <code>format</code> stops with an exception</summary>

✓ Right. The first invoice stops at `PROMPT.format` with `KeyError: '"vendor"'`, before there is any message to send, and so does every invoice after it. The traceback names a key that is nowhere in your code as a variable, so it takes a while to see that the example written for the model broke the run.

Building the prompt with `str.format` treats every pair of braces in the string as a slot, not only the ones you meant. The example's opening brace starts a slot named `"vendor"`, the text after its colon is read as formatting instructions, and `format` has no argument by that name, so it raises before the prompt exists. An f-string reads its braces the same way, and the same example breaks it with a `ValueError`.

Write literal braces doubled, `{{` and `}}`, or build the example with `json.dumps` and fill it in as one more value.

</details>

<details><summary>B. A request holding the invoice text and the example, both intact</summary>

✗ In Jinja and similar template engines a slot is written `{{ name }}` and a single brace is plain text, and the example's braces are single. `str.format` reads every single brace in the string: the example's `{` opens a slot called `"vendor"`, and with no value by that name, `format` raises a `KeyError` before the prompt is built.

</details>

<details><summary>C. A request with the example missing from its last line</summary>

✗ Template engines often render a missing slot as nothing, a quiet failure. `str.format` is not quiet about a slot it cannot fill: it stops with a `KeyError` for the name `"vendor"`, so there is no last line to be missing.

</details>

<details><summary>D. A request, and a reply that repeats the example's vendor and total</summary>

✗ A model handing back Vendor Name and 0.00 from the example is a real risk with examples in prompts. Here it never gets the chance: the run stops at the example's first brace, so there is no reply to repeat anything.

</details>

---

###### 5. Pasting a shoe shop's stock into the prompt as JSON

A shop that sells shoes has an assistant that answers questions about what is in stock. Your code loads the products from the database as a list of dicts and places them in the system prompt through an f-string, under a line that tells the model the list is JSON. One product has no price yet, and one name has an apostrophe in it.

```python
products = [
    {"name": "Hiking Boot", "price": 112.0, "in_stock": True},
    {"name": "Children's Rain Boot", "price": None, "in_stock": False},
]
system = f"Answer from this product list, given as JSON:\n{products}"
reply = call_model(messages=[{"role": "system", "content": system},
                             {"role": "user", "content": question}])
```

What text does the model get where the product list goes?

<details><summary>A. Valid JSON, with <code>true</code>, <code>false</code> and <code>null</code></summary>

✗ A Python dict and a JSON object share braces, colons and commas, and in an editor they are hard to tell apart. What an f-string makes of a list is Python's notation, with single quotes and `True`, `False` and `None`. Only `json.dumps` turns the list into JSON text.

</details>

<details><summary>B. Python's notation, with <code>True</code>, <code>False</code> and <code>None</code></summary>

✓ Right. Under the line the model finds `[{'name': 'Hiking Boot', 'price': 112.0, 'in_stock': True}, {'name': "Children's Rain Boot", 'price': None, 'in_stock': False}]`: most strings in single quotes, the name with the apostrophe in double quotes, and `True`, `False` and `None` in the places where JSON writes `true`, `false` and `null`. The prompt calls the list JSON, and no JSON parser would accept it.

Building the prompt through an f-string calls `format()` on each value to get its text, and the text of a list is its Python notation, the same you see when you print it in a Python shell. That notation picks quotes one string at a time and writes the constants as `True`, `False` and `None`. A prompt template fills its slots with whatever text a value gives, and an f-string knows nothing about JSON.

Write the list with `json.dumps(products)` to send JSON, so the prompt carries what the line above it claims.

</details>

<details><summary>C. Python's notation, with the empty price left out</summary>

✗ Serializers that skip empty fields are common, and the second product's price is `None`. An f-string writes every key the dict holds: the second product reads `'price': None` in the prompt, an empty price in plain view.

</details>

<details><summary>D. No text: the f-string raises on a list</summary>

✗ `"text" + products` raises a `TypeError`, and an f-string does the same job of putting a value into text. An f-string converts the value to text first, so any object goes into it, and a list arrives as its Python notation.

</details>

---

###### 6. Starting a new chat on a bank's support bot

A bank's support bot keeps each chat's messages in a list, and a new chat calls your function with the customer's first message and no history. The first customer of the morning asks when a transfer will arrive and gets an answer. Then a second customer opens a new chat and asks how to freeze a card.

```python
SYSTEM = {"role": "system", "content": "You are a bank's support assistant."}

def answer(text, history=[]):
    if not history:
        history.append(SYSTEM)
    history.append({"role": "user", "content": text})
    reply = call_model(messages=history)
    history.append({"role": "assistant", "content": reply})
    return reply
```

What goes out in the second customer's first request?

<details><summary>A. Only the system prompt and the card-freezing question</summary>

✗ The `if not history` line was written for a new chat, and `history` defaults to an empty list. The empty list was built once, when the `def` line ran, and the first chat filled it. The second call gets that same list back, already holding three messages.

</details>

<details><summary>B. The first customer's chat, then the system prompt again and the card-freezing question</summary>

✗ Once you see that the list is shared, the next worry is the `if not history` line firing for every new chat and adding the system prompt again. That line tests the list, which already holds three messages, so it is skipped, and the system prompt goes out once, at the top, with the first customer's exchange right below it.

</details>

<details><summary>C. The first customer's chat, then the card-freezing question</summary>

✓ Right. Four messages go out: the system prompt, the first customer's question about the transfer and the answer to it, and then the question about freezing a card. No exception is raised, and from then on each new chat's request carries every earlier customer's conversation, with whatever they typed about their cards and accounts, growing slower and dearer until it is too long for the model to take.

Python builds a default value once, when it runs the `def` line, and every call that leaves out the argument gets that same object. The first chat filled the list, and the second chat's `if not history` finds it full. The message list a call sends is whatever that list holds, so the model reads the earlier customer's exchange as part of this chat. A server with several worker processes keeps one such list in each, so a chat leaks into whichever later chats the same process takes.

Give the argument a default of `None` and create the list inside the function when it is missing, or keep each chat's history in a store keyed by the chat.

</details>

---

###### 7. Summarising weekend tickets in an async handler

A delivery firm summarises each new support ticket so the morning shift can skim them. Your handler is an async function that calls the model through `acall_model`. Overnight 140 tickets come in, and the handler runs once for each.

```python
async def on_new_ticket(ticket_id, body):
    summary = acall_model(messages=[{"role": "user", "content": "Summarise this ticket: " + body}])
    await save_summary(ticket_id, f"{summary}")
```

What has happened by morning?

<details><summary>A. A summary on every ticket</summary>

✗ In ordinary Python, writing a function's name with parentheses runs it. Calling an async function only creates a coroutine, and its body, with the request in it, never starts; each ticket gets the coroutine's text in place of a summary.

</details>

<details><summary>B. A request for each ticket, and a placeholder where each summary goes</summary>

✗ In JavaScript, calling an async function starts its work at once and hands back a promise, and Python uses the same `async` and `await` words. A Python coroutine is not started by being created: until something awaits it or makes it a task, it holds the call's arguments and nothing more, so no request leaves and each summary field gets the coroutine's text.

</details>

<details><summary>C. No requests, and an exception for every ticket</summary>

✗ Python raises at once on many call slips, such as a missing argument, and a forgotten `await` is the same kind of slip. Here nothing is raised: the coroutine is a valid value, the f-string turns it into text, and the handler finishes normally. Python prints only a `RuntimeWarning` when the unused coroutine is thrown away.

</details>

<details><summary>D. No requests, and a coroutine's text where each summary goes</summary>

✓ Right. No request went out all night, and the model saw none of the tickets. Each ticket's summary field holds the text `<coroutine object acall_model at 0x…>`; the handler finished every time, and the only trace in the logs is a single warning, printed once under Python's default settings, saying a coroutine was never awaited.

Calling an async function gives back a coroutine, and none of the function's body has run yet. The body, and the request inside it, run only when something awaits the coroutine or hands it to the event loop as a task. Here nothing does: the f-string turns the coroutine into its text, and when the object is thrown away, Python prints a `RuntimeWarning` that it was never awaited, which is not an exception and stops nothing.

Write `summary = await acall_model(...)`. Turning on asyncio's debug mode makes the warning show where each such coroutine was created.

</details>

---

###### 8. Adding await to a hotel chat's model call

A hotel's booking chat runs on an async web server. Its handler calls the model through `call_model`, a function of yours built on the provider's synchronous client, which waits for the reply and returns its text. So that other guests are not kept waiting, a colleague adds `await` before the call and changes nothing else.

```python
async def handle(message):
    reply = await call_model(messages=[{"role": "user", "content": message}])
    return reply
```

What happens when a guest sends a message?

<details><summary>A. The reply arrives, and other guests are served during the wait</summary>

✗ In every async example you have read, `await` makes the line wait while the server serves others. That works only on something `await` can wait on, such as a coroutine, and here the blocking call runs to the end first, holding the thread; then `await` gets a plain string and raises a `TypeError`.

</details>

<details><summary>B. The model answers, and then an exception replaces the reply</summary>

✓ Right. The guest's question reaches the model and its reply comes back into your process; then the `await` line raises a `TypeError`, a string being something it cannot await, and the guest sees an error page. By then the call has been answered and billed in full.

Python works out the expression after `await` first, and `call_model(...)` is a blocking call: it sends the request, keeps the thread busy until the reply is in and returns a string. `await` inspects what it was given only after that, and a string is not something it can wait on, so it raises. The keyword makes nothing asynchronous; it waits on something that already is, such as the coroutine an async function returns.

Send the call through the provider's async client, or pass the blocking call to `asyncio.to_thread`, so it does not hold the event loop.

</details>

<details><summary>C. The reply arrives, the same as before the edit</summary>

✗ The wrapper worked before the edit, and the call itself still goes out and comes back. Python's `await` accepts only awaitables, such as a coroutine or a task. Given the string the call returned, it raises a `TypeError`, and the reply never reaches the guest.

</details>

<details><summary>D. An exception, and no request ever sent</summary>

✗ Python checks plenty before it runs anything, from syntax to indentation. This one shows up only at run time: the call inside runs first and sends the request, and the `TypeError` comes afterwards, when `await` finds a string.

</details>

---

###### 9. A museum notice in Italian, Japanese and Portuguese

A museum translates each new notice for visitors into Italian, Japanese and Portuguese, inside a web service that keeps running. One morning the Italian call fails with a connection error after a fifth of a second. The other two calls would take about three seconds each.

```python
async def translate(notice):
    try:
        italian, japanese, portuguese = await asyncio.gather(
            acall_model(messages=ask("Italian", notice)),
            acall_model(messages=ask("Japanese", notice)),
            acall_model(messages=ask("Portuguese", notice)),
        )
    except ConnectionError:
        return None
    return {"it": italian, "ja": japanese, "pt": portuguese}
```

What happens to the Japanese and Portuguese calls once the Italian one has failed?

<details><summary>A. Neither of them is ever sent</summary>

✗ The three calls are written one under another, and code written in order usually runs in order. `gather` starts all three together, so by the time the Italian call fails, the Japanese and Portuguese requests are already on their way, and they run to the end.

</details>

<details><summary>B. Both are cancelled with the Italian call</summary>

✗ Stopping the rest when one call fails is what you would want, and a `TaskGroup` does exactly that. `gather` does not: with its default settings it passes the first exception on and leaves the other calls running. Both finish about three seconds in, and both are billed.

</details>

<details><summary>C. Both finish, and no code ever reads their replies</summary>

✓ Right. After a fifth of a second, `translate` has already returned `None`; near the three-second mark the Japanese and Portuguese replies arrive and land nowhere, each one paid for.

With its default settings, `asyncio.gather` passes the first exception straight up to whoever awaits it and lets the other calls carry on, uncancelled. Your `except` returns while the Japanese and Portuguese calls are still on their way, and when they finish, their results go to a `gather` nobody is awaiting any more.

Pass `return_exceptions=True` to get every result, errors included, in one list and keep the translations that worked, or use a `TaskGroup`, which cancels its other tasks as soon as one fails and raises the errors as an `ExceptionGroup`, caught with `except* ConnectionError`.

</details>

<details><summary>D. Both finish, and <code>translate</code> returns them with Italian missing</summary>

✗ Getting back what worked, with a gap where Italian failed, is possible with `gather` when you pass `return_exceptions=True`. These calls use the default, so the first exception goes straight to your `except`, `translate` returns `None`, and the two translations that arrive later have nowhere to go.

</details>

---

###### 10. A ten-second limit on a car-rental chat

A car-rental chat gives the model ten seconds to answer. `acall_model` raises `ConnectionError` when the network fails, and the client library's own time limit is two minutes, well past those ten seconds. A late answer should bring a promise of a text message, a network failure a note that the customer seems to be offline. One afternoon, replies take about twenty-five seconds.

```python
async def answer(question):
    try:
        async with asyncio.timeout(10):
            return await acall_model(messages=[{"role": "user", "content": question}])
    except OSError:
        return "You seem to be offline. Please try again once you're connected."
    except TimeoutError:
        return "This is taking longer than usual. We'll text you the answer."
```

What does a customer read after ten seconds?

<details><summary>A. The promise of a text message</summary>

✗ You wrote a clause for exactly this failure, and its name matches the error. Python does not look for the best match: it takes the first clause that fits, and `TimeoutError` is a kind of `OSError`, so the clause above wins every time.

</details>

<details><summary>B. The note that they seem to be offline</summary>

✓ Right. Ten seconds in, the customer is told they seem to be offline and should try again once connected. The promise of a text never appears, on this timeout or on any later one.

Once the ten seconds are up, `asyncio.timeout` cancels the call and puts Python's own `TimeoutError` in its place, and since Python 3.3 `TimeoutError` has been a kind of `OSError`. Python checks `except` clauses from the top and runs the first whose class matches, parents included, so the `OSError` clause takes every timeout, the clause below it can never run, and the module loads without a warning. A client library raises its own error classes when its own time limit runs out, and where they sit in the class tree is the library's choice.

Put the narrower class first, and check where a client library's classes sit before you order clauses around them.

</details>

<details><summary>C. An error from the chat, which neither clause catches</summary>

✗ Inside, `asyncio.timeout` works by cancelling the call, and a cancellation error is caught by neither clause. The context manager catches that cancellation itself and raises a `TimeoutError` in its place, which the first clause, `OSError`, handles.

</details>

<details><summary>D. Nothing: Python refused to load the module, and the chat is down</summary>

✗ Java refuses to compile a catch clause that an earlier one already covers, and this clause is covered in the same way. Python loads the module silently and checks the clauses in order at run time, so the `OSError` clause answers every timeout and the `TimeoutError` clause never runs.

</details>

---

###### 11. Reporting an outage after three tries in a nightly job

At a logistics company, a nightly job writes a delivery report with the model. `call_model` raises `ConnectionError` when the provider cannot be reached. The job tries each call three times and, if all three fail, raises an error that names the last failure as its cause. One night the provider is down for an hour.

```python
def ask_with_retries(messages):
    for attempt in range(3):
        try:
            return call_model(messages=messages)
        except ConnectionError as err:
            time.sleep(2 ** attempt)
    raise RuntimeError("model call failed 3 times") from err
```

Which error does the next morning's report show?

<details><summary>A. A <code>RuntimeError</code> whose cause is the last <code>ConnectionError</code></summary>

✗ The loop ran three times and `err` was set every time. Python deletes the name as each `except` clause ends. By the `raise` line it is gone, and that line raises an `UnboundLocalError` before the `RuntimeError` can be built.

</details>

<details><summary>B. A <code>RuntimeError</code> that names no cause</summary>

✗ Clearing a variable usually means setting it to `None`, and `raise ... from None` gives an error with no cause. Python does not set the name to `None`; it deletes it. Reading the deleted name is what fails, so even a `from None` would never be reached.

</details>

<details><summary>C. An <code>UnboundLocalError</code>, with the <code>ConnectionError</code> printed above it</summary>

✗ Python's tracebacks often show one error above another, "during handling of the above exception", and three `ConnectionError`s were handled here. That chaining happens only for an error raised inside an `except` clause. The `raise` line runs after the loop, with no exception being handled, so the `UnboundLocalError` comes alone.

</details>

<details><summary>D. An <code>UnboundLocalError</code> alone, with no <code>ConnectionError</code></summary>

✓ Right. The morning's report holds one error, an `UnboundLocalError` on the `raise` line about the local variable `err`, and nothing that names the `ConnectionError`, so nothing in the report points at the provider's outage.

When an `except` clause binds its exception with `as err`, Python deletes the name `err` as the clause ends, so that the exception's traceback does not keep the function's variables alive. After the loop the name no longer exists, and the `raise` line fails before it can build the `RuntimeError`. That line sits after the loop, where Python is handling no exception, so there is nothing to chain.

Keep the last failure in a name of your own: copy it with `last_error = err` inside the clause, and name `last_error` as the cause in the final `raise`.

</details>

---

###### 12. Typed invoice records in a bookkeeping service

A bookkeeping service turns supplier emails into invoice records. `call_model` returns the model's JSON reply as text, and your code builds an `Invoice` from it, a `dataclass` whose `total` field is typed as a float. On the 3rd of the month, one reply gives the total as the string `"865.40"` rather than as a number, and the books are totalled on the 31st.

```python
@dataclass
class Invoice:
    supplier: str
    total: float

invoices = []

def ingest(reply):
    invoices.append(Invoice(**json.loads(reply)))

def month_end_total():
    return sum(invoice.total for invoice in invoices)
```

What follows from that string total?

<details><summary>A. The string is kept, and the 31st's sum fails</summary>

✓ Right. The 3rd passes quietly: `ingest` keeps an `Invoice` whose total is `'865.40'`, a string. The trouble waits for the 31st, when `month_end_total` raises `TypeError` as it adds that string to a number, four weeks after the reply that caused it, and no total comes out.

A type hint tells readers and tools what a field is meant to hold; Python itself never checks it while the program runs. A `dataclass` builds a constructor that stores whatever it is given, with no check and no conversion, so the string passes through `ingest` untouched and fails only when arithmetic meets it.

Checking the reply has to be code that runs: test or convert each field as the record is built, in `__post_init__` for example, or use a validation library that turns `"865.40"` into a number or refuses it.

</details>

<details><summary>B. <code>ingest</code> refuses the reply on the 3rd with a <code>TypeError</code></summary>

✗ Typed code in a compiled language refuses a string where a float is declared, and `total: float` makes the same promise on the page. Python stores the hint and never checks it at run time, and a `dataclass` adds no check of its own. The string goes into the record on the 3rd without a word.

</details>

<details><summary>C. The record holds 865.4, and the 31st's sum includes it</summary>

✗ `float("865.40")` turns the text into 865.4, and the field is typed as a float. A `dataclass` converts nothing, so the field holds the string `"865.40"`, and the sum on the 31st fails on it.

</details>

<details><summary>D. The 31st's sum is a string of digits glued together</summary>

✗ In JavaScript, adding a number to a string glues them together as text. Python's `+` does not convert: a float plus a string raises a `TypeError`, so the sum on the 31st stops with an error.

</details>

---

###### 13. A refund decision from an electronics shop's returns bot

An electronics shop runs a returns bot that asks the model whether an order qualifies for a refund, and your prompt asks for JSON with `refund`, true or false, and a `reason`. Returns are accepted within 30 days. For an order bought 40 days ago, the reply reads `{"refund": "false", "reason": "Purchased 40 days ago; the window is 30 days."}`, with the false in quotes.

```python
decision = json.loads(reply)
if decision["refund"]:
    issue_refund(order_id)
else:
    send_message(order_id, decision["reason"])
```

What happens to the order?

<details><summary>A. No refund, and the customer is sent the reason</summary>

✗ The word says false, and a person reading the reply would stop the refund. Python's `if` tests truth, and it does not read words: an empty string is false and every other string is true, `"false"` included. The test passes and the refund goes out.

</details>

<details><summary>B. A refund is issued</summary>

✓ Right. The customer gets their money back, 40 days after buying, and never hears the reason the model wrote; every refusal written with its false in quotes turns into a refund.

Parsing keeps JSON's types: `false` becomes `False`, and `"false"` becomes the five-letter string `'false'`. Acting on the reply with `if decision["refund"]:` tests the value's truth, and every non-empty string is true in Python, whatever its letters say. The model was asked for true or false and wrote a string, a slip no parser catches.

Check the type before you act, with `decision["refund"] is True`, or hold the reply to a schema that types the field as a boolean, and send anything else to a person.

</details>

<details><summary>C. Neither: the code raises an exception</summary>

✗ Python refuses to add text to a number, and here a string sits where a yes or no belongs. An `if` accepts any value at all and tests its truth. The non-empty string `"false"` is true, so the refund branch runs and nothing is raised.

</details>

---

###### 14. Rerunning a help centre's summary job on its saved cache

A help centre shows a short summary above each of its 420 articles. Your job summarises each article once and keeps the summaries in a JSON file keyed by the article's id, an integer, so a rerun pays only for new articles. Monday's run summarises all 420. On Tuesday the job runs again over the same 420 articles.

```python
def summarise_all(articles):
    try:
        with open("summaries.json") as file:
            cache = json.load(file)
    except FileNotFoundError:
        cache = {}
    for article in articles:
        if article.id not in cache:
            cache[article.id] = call_model(messages=summary_request(article))
    with open("summaries.json", "w") as file:
        json.dump(cache, file)
```

What does Tuesday's run do?

<details><summary>A. Every summary is found in the file, and no call goes out</summary>

✗ A file written with `json.dump` and read back with `json.load` gives back what you wrote for most data, and Monday's file opens with every id in place. Keys are the exception: JSON keys are text, so the integer ids come back as strings, and Tuesday looks up integers, finds none and sends all 420 articles to the model again.

</details>

<details><summary>B. The run ends with an error when it saves the file</summary>

✗ JSON keys must be strings, and these keys are integers. Python's `json.dump` turns them into strings, and Monday's file saves cleanly. The trouble starts on Tuesday, when the keys come back as strings and no integer id matches.

</details>

<details><summary>C. Every article goes to the model again, and the file saves</summary>

✓ Right. On Tuesday every article goes to the model again, all 420 of them, and neither day raises an error; every run pays for 420 summaries, and the cache never saves a call.

`json.dump` writes every key as a string, so article 17 is saved under `"17"`, and `json.load` gives back the string `'17'`. Tuesday's `article.id not in cache` asks about the integer 17, the cache holds only the string, and every lookup misses: storing the results as JSON changed the keys' type on the round trip. After the run the dict holds both the string `'17'` from the file and the integer 17 added on Tuesday, so the file written from it carries `"17"` twice. The next load keeps the last of the two, and the misses start over.

Convert the id to a string when you write and when you look up, with `str(article.id)`.

</details>

---

###### 15. Counting a handbook's chunks before embedding them

An HR team runs an assistant that searches the staff handbook. Every night a job empties the search store, cuts the handbook into chunks, has a model embed each one and adds it to the store, and `chunks_of` is a generator. Before the loop you added a log line with the number of chunks, and tonight's log says 186.

```python
def chunks_of(handbook):
    for section in handbook.sections:
        yield from split_into_chunks(section.text)

chunks = chunks_of(handbook)
log.info("indexing %d chunks", len(list(chunks)))
for chunk in chunks:
    store.add(chunk, embed(chunk))
```

What does the store hold after tonight's run?

<details><summary>A. Nothing from the handbook, and a run that ends cleanly</summary>

✓ Right. Not one chunk reaches the store, and the run ends as if all went well: `embed` never runs, though the log line reports 186 every night, and the assistant answers staff with no handbook behind it.

A generator hands out its values once. `list(chunks)` drains it to count the chunks, so when the loop asks that same generator for values, there are none left, which ends a `for` loop quietly, as an empty list would.

Take a generator's values in one pass: make a list once, `chunks = list(chunks_of(handbook))`, and count and loop over that, or count inside the loop as you go.

</details>

<details><summary>B. Every chunk, each with its embedding</summary>

✗ Counting a list before looping over it never loses anything, and `len(list(chunks))` counts a list. `chunks` is a generator, though, and the count used it up: the loop asks for more, gets none, and the 186 chunks never reach `embed`.

</details>

<details><summary>C. Nothing, and the run stops with an exception at the loop</summary>

✗ Asking a finished generator for another value raises `StopIteration`, an exception. A `for` loop takes `StopIteration` as its signal to end. On a used-up generator it ends at once, raises nothing, and the job finishes with an empty store.

</details>

---

###### 16. Recording token usage after answering on the command line

At a translation agency, staff ask a model questions from a command-line tool, and each run should leave one row of token usage in a database table for the monthly bill review. Here `acall_model` returns two values, the answer and the call's token usage. `save_usage_row` writes the row through an async database client, and to print the answer without waiting on it, your code starts the write as a task.

```python
async def main(question):
    answer, usage = await acall_model(messages=[{"role": "user", "content": question}])
    print(answer)
    asyncio.create_task(save_usage_row(usage))

asyncio.run(main(sys.argv[1]))
```

After a week of runs, what is in the usage table?

<details><summary>A. A row for each run</summary>

✗ Starting a task is the usual advice for slow side work, so the write can finish in the background. The task needs the event loop to run, and the loop lasts only as long as `main`: `main` returns at once, `asyncio.run` cancels what is left, and the row is never written.

</details>

<details><summary>B. Rows for some runs and none for others, changing from run to run</summary>

✗ The database write and the program's exit race each other, and a race is the textbook case of results that vary from run to run. This race has one outcome on every run: `main` returns before the write can finish, and `asyncio.run` cancels the task before closing the loop.

</details>

<details><summary>C. No rows, and a warning at the end of each run</summary>

✗ Python warns about tasks left behind, with a line saying a task was destroyed while pending. `asyncio.run` avoids that case: it cancels the task and waits for the cancellation, so it closes the loop with nothing pending and prints nothing.

</details>

<details><summary>D. No rows, and no output but the answer</summary>

✓ Right. Not one row arrives. Each run prints its answer and exits, and the write it started is cancelled before it reaches the database, so at the bill review the usage table is empty.

A task gets time only while its event loop is running. Once `main` returns, `asyncio.run` cancels whatever tasks are left before shutting the loop, so the write is cancelled at its first wait and no exception reaches you. Creating a task asks for something to run alongside; it does not promise that it finishes.

Await the write before `main` returns, with `await save_usage_row(usage)`, or keep your tasks in a set and await them all before the end.

</details>

---

## Further reading

- [An Empirical Study on Challenges for LLM Application Developers](https://arxiv.org/abs/2408.05002), 2024: 29,057 questions from a model provider's developer forum, sorted into a taxonomy of what goes wrong when building on a model's API, from prompts to API calls and limits.
- [Prompts Are Programs Too!](https://arxiv.org/abs/2409.12447), 2024: a study of 20 developers who build software around prompts, who reason about how the model behaves more than about their code and struggle to make that picture reliable.

Prev: [Cost and latency](../cost-and-latency/README.md) · Next: [Streaming](../streaming/README.md)
