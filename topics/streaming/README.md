# Streaming

Needs: [Tool calling](../tool-calling/README.md) · [Cost and latency](../cost-and-latency/README.md) · Reviewed October 2026

Streaming sends a model's reply in pieces while the model is still writing it: one request, one open connection, and a series of events until the model stops. Your code reads the stream one event at a time, and an event can carry more than text.

- **Token**: a piece of text, often a word or part of one; the model writes a reply one token at a time, and limits and bills are counted in tokens.
- **Event**: one message in a stream. In the common format, called server-sent events, an event is one or more lines of text, among them a `data:` line with a JSON object on it, and a blank line closes it; some servers you run yourself send one JSON object per line instead.
- **Idle limit**: the longest a connection may stay silent, with no bytes arriving, before a client, a proxy or a network closes it; client libraries often call it a read timeout. A deadline, by contrast, limits the whole call, from the request to the last byte.
- **Finish reason**: the field that says why a reply ended, such as a natural end, a stop sequence (a string the request tells the model to stop at), the generation cap (the most tokens the request lets the reply run to) or a tool call.

A provider's API streams when the request asks for it, as server-sent events whose kinds, names and contents differ by provider; on your own server, the server's settings decide details such as how many tokens one event carries.

The kinds of event a stream can hold:

| Event | What it does | Where you find it |
| --- | --- | --- |
| Start | opens the reply, with details such as its id and the model | some provider APIs |
| Text piece, often called a delta | adds the next part of the reply's text | every provider's API and your own server |
| Tool call | carries a call the model makes to one of your tools | provider APIs and your own server, when tools are on |
| Error | reports that the call has failed | some provider APIs |
| End | closes the reply and the stream | every provider's API and your own server |

While the model is silent, some systems also send keep-alive events, which carry nothing and keep the connection from looking idle; and a model that reasons before it answers can stream that reasoning as pieces of its own, apart from the text.

The problems sit in the code between the connection and the screen: the loop that reads each event, the limits set around the call, and what your code shows, saves and sends again when a stream breaks partway.

This topic lets you:

- judge what streaming will change for a slow chat, a long document or a long report, and what it does to the time limits around a call
- write the loop that reads a stream, from its text to a tool call, a failure and the way the reply ended, and the parser under it when you read the bytes yourself
- decide what your code shows, checks, saves and resends while a reply is still arriving or after it breaks halfway

###### 1. Turning on streaming for a slow plant-care chat

At a garden centre, customers of the plant-care chat watch a spinner for about twelve seconds before an answer appears in full; the chat runs on a provider's API. Assume most of that time is the model writing the answer, which runs to several paragraphs. The store manager asks for streaming to speed the chat up, and you change nothing else: the model and the prompts stay as they were.

After the switch, how long after asking does a customer see the last word of the answer?

<details><summary>A. Several seconds less than the twelve before</summary>

✗ Streaming tops most advice on slow chats, and a fix for slowness is naturally read as a fix for the whole call. It changes when the reply leaves the server, not how fast the model writes it. The model still writes one token after another at the same pace, and the last paragraph is ready about twelve seconds in, as before.

</details>

<details><summary>B. A moment more than the twelve before</summary>

✗ Where one response used to come back, hundreds of small events now arrive, and it is fair to guess that the extra traffic adds some time. Each event leaves the moment its piece is written and rides the connection opened for the request. The final event goes out right after the model's last token, at about twelve seconds, as the whole response did before.

</details>

<details><summary>C. About twelve seconds, as before</summary>

✓ Right. Words begin to show a moment after the customer asks, and the whole answer still lands about twelve seconds in. The model, its pace and the number of tokens it writes have not changed.

Customers start reading within a moment instead of staring at the spinner, so the chat feels quicker. The weekly report on answer times, which clocks each answer at its final word, still shows twelve seconds, and the manager who asked for a faster chat finds no gain in it.

Without streaming, the server holds the answer until its last token and sends it whole; with streaming, each piece leaves the server the moment the model produces it. The time to first token falls, and the total time stays where it was, since the model writes one token after another either way. Picture a server that reads 10,000 input tokens a second and writes 50 reply tokens a second: a 1,500-token prompt and a 600-token answer give the first words at 0.17 seconds and the whole answer at 12.15 seconds, with streaming or without. Streaming leaves the bill alone as well: the call reads and writes the same tokens, and tokens are what you pay for. If the whole answer has to come sooner, the reply must get shorter or the model faster; streaming changes only how the wait feels. A server you run yourself streams the same way, and both times behave there as they do here.

</details>

---

###### 2. Streaming a story writer and a contract reader

A publisher runs two tools on a provider's API, and both show nothing until a reply is complete. The story tool turns a one-line idea into a 1,500-word story in about 40 seconds. The contract tool answers a question about a 300-page contract in two sentences and takes about 19 seconds. Asked the same question about a one-page letter, it answers in about a second. You turn on streaming for both.

After the switch, when does each tool show its first words?

<details><summary>A. Both tools almost at once</summary>

✗ Streaming is the standard answer to a long wait, and both tools have one. It lets each piece leave as soon as it is written, and the contract tool writes nothing until the model has read all 300 pages. That reading is most of its 19 seconds, so its first words still come late.

</details>

<details><summary>B. The story tool at once, the contract tool after most of its wait</summary>

✓ Right. The story tool's first words appear almost at once, while the contract tool's still take most of its 19 seconds. Both tools still finish at the same moment as before the switch.

Writers using the story tool watch their story start within a moment instead of facing a blank panel for 40 seconds. Lawyers using the contract tool get the same long blank wait, then two sentences that appear almost together, and their complaints about speed keep coming.

The time to first token covers reading the whole request and writing the first token, and streaming changes nothing before that first token exists. A model reads all of its input before it writes the first token of the reply, and a longer input takes longer to read. Picture a server that reads 10,000 input tokens a second and writes 50 reply tokens a second. The story tool reads 300 tokens in 0.03 seconds and spends 40 seconds writing 2,000, so its first words move from 40.03 seconds to 0.05. For the contract tool, the 180,000 tokens of the contract take 18 seconds to read and the 60-token answer 1.2 to write, so its first words come at 18.02 seconds instead of 19.20. Streaming helps a tool whose wait is writing. A tool whose wait is reading needs a shorter input, such as only the sections of the contract the question is about. Your own server reads the same way, and some servers ping the connection during that silent reading so that it does not look idle.

</details>

<details><summary>C. The contract tool at once, the story tool after most of its wait</summary>

✗ The contract tool has the long, silent wait that streaming is meant to fix, and a two-sentence answer should be quick to start. The silence is the reading, as the one-page letter shows: it answers in about a second, so the 300-page contract spends about 18 seconds being read before the model writes its first token. Streaming cannot send a piece that does not exist yet.

</details>

<details><summary>D. Both tools after the same wait as before</summary>

✗ Both tools send the same tokens as before, so it is tempting to expect the same waits. The total time stays, but the first piece of the story moves: the story tool's 40 seconds are almost all writing, and streaming sends each part of the story as soon as it is written, from the first moment on.

</details>

---

###### 3. Keeping the reply check after turning on streaming

A bank's support chat runs every reply through a check for promises the bank cannot make, such as a guaranteed refund, and shows a safe reply in place of one that fails. To make the chat feel quicker, you turn on streaming and show each piece of text as it arrives. The check is unchanged: it runs on the reply once your code has it, and swaps a failing one for the safe reply.

When a reply promises a refund, what does the customer see?

<details><summary>A. The whole reply, promise included, then the safe reply</summary>

✓ Right. The customer watches the whole reply arrive, promise included, and then sees it replaced by the safe reply. The check ran, after the last piece was already on screen.

A customer told that a full refund is guaranteed by Friday takes a screenshot before the text changes. The complaint that reaches the bank a week later quotes the promise word for word, while the bank's own log records that reply as blocked.

Once your code shows each text piece as it arrives, the customer reads the reply while it is being written, and a check on the finished reply runs after every piece has been shown. A check can stop only what has not been shown yet. To keep it in front of the customer, hold the text back and check each sentence or paragraph before you show it, which gives back part of the speed; or keep the whole-reply check and show the replies that need it only once they pass. Showing and checking happen in your code, so the outcome is the same behind a provider's API and on your own server.

</details>

<details><summary>B. Only the safe reply, a little later than usual</summary>

✗ The check worked before streaming, and nothing in it changed, so it seems to stand between the model and the customer still. It no longer does: each piece goes to the screen when it arrives, and the check waits for the finished reply. By the time it runs, every word of the promise has been on screen.

</details>

<details><summary>C. The reply up to the promise, never the promise itself, then the safe reply</summary>

✗ A check that sits beside the stream can be pictured watching it, ready to step in at the bad sentence. This one reads only the finished reply, after the last piece. Every piece, the promise included, has been shown by then, and the swap comes after the customer has read it all.

</details>

---

###### 4. Streaming long reports past two time limits

For an analytics firm, an assistant writes long market reports, each about five minutes from the request to its last word. Between your service and the provider's API sits a proxy with an idle limit of 60 seconds, and your worker sets a deadline of 3 minutes on each call. Before streaming, every report failed a minute in. You turn on streaming.

What does an analyst get after the switch?

<details><summary>A. The whole report, five minutes in</summary>

✗ Streaming is the usual fix for long replies that die on a timeout, and it does fix one of the two limits here. Pieces now arrive while the model writes, so the proxy never sees a quiet minute. Your worker's limit is a different kind: it counts from the request, and at three minutes it stops reading, two minutes short of the end.

</details>

<details><summary>B. No report at all, cut off a minute in as before</summary>

✗ One request is still open for five minutes, and a proxy that cut it before might be expected to cut it again. This proxy watches for silence, not length. With streaming, bytes move for as long as the model writes, so the silent minute never comes, and the report keeps arriving until your worker's own limit stops it.

</details>

<details><summary>C. Three minutes of the report, then nothing</summary>

✓ Right. The report streams for three minutes and stops partway, when your worker stops reading. The proxy never closes the connection, which is never quiet long enough.

Analysts get reports that break off a little past halfway, often in the section they asked for. One that ends between two sentences can pass for a finished report with a reader in a hurry, and it goes into a client deck that way.

The two limits measure different things. An idle limit closes a connection that has gone quiet, and streaming keeps it busy: bytes move whenever the model writes a piece, and some systems send keep-alive events in quiet stretches too. A deadline counts from the request to the last byte, and streaming leaves that time as it was, since the report still takes five minutes to write. Here your worker stops at 180 seconds of the report's 300, with 60% of it read. Set the deadline above the longest report you expect, or keep a short limit on silence and a much longer one on the whole call. Treat a report as complete only when its end event has arrived, so a cut report is never saved as a finished one. The proxy and the worker are yours, so it makes no difference whether the model runs behind a provider's API or on your own server.

</details>

<details><summary>D. One minute of the report, then nothing</summary>

✗ A limit of 60 seconds can be taken for the longest a connection may stay open, and a five-minute stream would then reach it a minute in, with part of the report shown. An idle limit counts silence only: every piece that arrives starts its count again, and pieces keep arriving while the model writes, so the proxy lets the stream run on.

</details>

---

###### 5. Running tools from a streamed reply

On a travel app, the assistant now streams its replies from a provider's API. `call_model` is your thin wrapper around the provider's client library; with `stream=True` it yields events, and `event.kind` says what an event brings: `"text"` with `event.text`, or `"tool_call"` with the tool's name in `event.name` and the arguments text in `event.arguments`. Until the switch, your code waited for the complete reply and took each tool call from it. A user asks for the weather in Lisbon, and the model calls `get_weather`.

```python
for event in call_model(messages=messages, tools=tools, stream=True):
    if event.kind == "text":
        show(event.text)
    elif event.kind == "tool_call":
        args = json.loads(event.arguments)
        run_tool(event.name, args)
```

What does this loop do with the call?

<details><summary>A. A <code>JSONDecodeError</code>, and no run of <code>get_weather</code></summary>

✓ Right. `json.loads` raises `JSONDecodeError` as soon as the first tool-call event arrives, so `get_weather` is never called. The arguments text on that event is only their beginning, empty or a few characters, and that is not valid JSON.

Any request that makes the model reach for a tool now crashes the turn, though replies in plain text keep streaming as before. Travellers asking about the weather, a flight or a hotel all hit it, and the trace shows an empty string where the arguments were expected.

A tool call streams the way text does. The event that opens it brings the tool's name, and the arguments follow as fragments of text over several events. Here `json.loads('')` raises "Expecting value", and `json.loads('{"city": "Lis')` raises "Unterminated string". Keep the fragments of each call, join them in order, and parse once the call is complete: the joined `{"city": "Lisbon"}` parses to the city. A reply with several calls tags each fragment with the call it belongs to, and some APIs send the whole call again in a closing event that you can run instead. A server you run yourself may send each call whole once it has picked the call out of the model's text, or in pieces; joining the fragments and parsing at the end works either way.

</details>

<details><summary>B. One run of <code>get_weather</code>, for Lisbon</summary>

✗ Your code used to find each call complete in the finished reply, and turning on streaming should change only when things arrive, not their shape. A provider's API streams a call's arguments the way it streams text, a fragment per event. The call's first event holds an empty or partial string, `json.loads` fails on that string, and `get_weather` is never reached.

</details>

<details><summary>C. Several runs of <code>get_weather</code>, the city spelled out further each time</summary>

✗ Some client libraries hand you a call's arguments as a growing snapshot, parsed so far, and each event could be pictured as one. These events carry fragments of text, not snapshots: one may hold `{"city": "Lis` and nothing more. That is not JSON, and neither is the empty first fragment, so the loop fails before any run.

</details>

<details><summary>D. The arguments shown in the chat as text, and no run</summary>

✗ A model writes a tool call as text, and a stream of text could be expected to show it like any other words. A provider's API finds the call in what the model writes and sends it as tool-call events, so none of it reaches `show`. The loop takes the first such event to `json.loads`, which raises on the empty opening fragment.

</details>

---

###### 6. A provider overload in the middle of a reply

A delivery company answers tracking questions in a chat that streams replies from a provider's API. `call_model` returns a response with its HTTP status in `response.status`, and `response.events()` yields the events, each with a `kind` such as `"text"` or `"error"`. One evening the provider is overloaded for twenty minutes, and some customers are partway through a reply when it starts.

```python
response = call_model(messages=messages, stream=True)
for event in response.events():
    if event.kind == "text":
        show(event.text)
if response.status != 200:
    show("The assistant is busy. Please try again in a minute.")
    log.warning("provider busy")
else:
    log.info("reply ok")
```

What does a customer see when the overload hits a reply that has already begun?

<details><summary>A. The rest of the reply, a little slower</summary>

✗ A reply already on its way feels accepted, as if a busy provider would finish it, only more slowly. A provider under load can fail a reply partway, sending an error inside the stream. This loop passes over every event that is not text, so the customer gets no more of the reply.

</details>

<details><summary>B. Part of the reply, then the busy message</summary>

✗ The status is read after the loop, once the overload has struck, so it seems bound to report it. The status comes once, as the stream opens, and it was 200. The overload arrives later as an event of kind `"error"`, which the loop passes over, so the busy message is never shown.

</details>

<details><summary>C. The reply again from its first word</summary>

✗ A client library can retry a failed call on its own, and a reply starting over from the top is what such a retry would show. Your wrapper is thin, and this code sends nothing again. No more text follows the error event, and the customer is left with the part already shown.

</details>

<details><summary>D. Part of the reply, then nothing more</summary>

✓ Right. The customer sees the reply stop partway, with nothing after it, not even the busy message. Your log records the turn as `reply ok`, like every reply that finished.

For twenty minutes, customers asking where a parcel is get half a sentence and a chat that has gone quiet. The error rate on your dashboard stays flat, since every one of those replies was logged as fine, and support hears about the evening from customers.

The HTTP status is sent once, before the first event, and all it can say is that the stream opened. A failure after that can only show inside the stream: as an error event of its own on an API that sends one, or as a stream that stops with no end event. Handle it in the loop: on an error event, show the busy message under the part already shown, offer to try again, and log the failure. Treat a reply as finished only when its end event arrives, and as failed when an error event comes or the stream closes without an end event, whatever the status said.

</details>

---

###### 7. Noting a summary cut by the generation cap

An insurer's portal streams a summary of each policy document onto the page, and you want a note under any summary the generation cap cut short. With `stream=True`, `call_model` yields events; the first event of every stream has the kind `"start"` and carries `event.reply`: the reply as it stands when the stream opens, with an `end_reason` field among its details. Then a long policy arrives, and the cap stops its summary partway.

```python
end_reason = None
for event in call_model(messages=messages, stream=True):
    if event.kind == "start":
        end_reason = event.reply.end_reason
    elif event.kind == "text":
        show(event.text)
if end_reason == "cap":
    show_note("This summary was cut short.")
```

What does the reader see under the cut summary?

<details><summary>A. The note saying the summary was cut short</summary>

✗ A web response sends its status and headers before the body, and the start event looks like the same thing: the reply's details first, the text after. But the start event goes out before the model has written anything, when nothing is known about how the reply will end. Its finish reason is empty, and the check after the loop finds nothing to match.

</details>

<details><summary>B. The cut summary alone, with no note under it</summary>

✓ Right. Nothing appears under the summary, which stops mid-sentence. After the loop, `end_reason` is still the empty value taken from the start event, so the check never matches.

A reader trusts the summary as complete, and the exclusions written after the cut never reach anyone who relies on it. Months later a claim is refused under a clause no summary mentioned, and the claims team blames the summaries rather than the cap.

The finish reason is known only once the model stops, so a stream reports it near the end, never at the start. The start event carries the reply as it stands before any text, with the finish reason empty; the reason itself comes in an event after the last piece of text. Read it there and branch on it after the loop. A summary cut by the cap streams like any other and ends normally, so the finish reason is the only sign of the cut. Your own server reports it the same way, in its last event.

</details>

<details><summary>C. The cut summary, then an error raised when the cap is reached</summary>

✗ Some limits end a call in an error, such as a request too long for the window, and the cap is a limit as well. But reaching the cap is a normal way for a reply to end. The stream closes like any other, with its finish reason set to the cap and nothing raised, so the loop runs to its end and the check after it finds the empty value.

</details>

---

###### 8. Parsing the event stream in a forwarding service

On a site that books tickets, the chat runs through your small forwarding service, which reads a model's stream with no client library and passes each event on to the browser. `reads` yields one read at a time: whatever bytes have arrived from the connection since the last one. The stream uses the common format: each event is a `data:` line holding a small JSON object, then a blank line. Against a test server on your laptop, every reply came through whole. In production, the stream reaches you over the internet.

```python
for data in reads:
    text = data.decode("utf-8")
    if text.startswith("data: "):
        on_event(json.loads(text[len("data: "):]))
```

What does the service do with production traffic?

<details><summary>A. Handles every event, as on the laptop</summary>

✗ The laptop test passed on every reply, since a local server's events came one to a read. Over the internet the bytes come in whatever sizes the network delivers, so a read can hold part of an event or two events together. Neither is one JSON object, and `json.loads` raises on both.

</details>

<details><summary>B. Raises a decoding error on replies in other languages only</summary>

✗ A test that passes on a laptop and fails with real users is so often an encoding story that a decoding error is the first guess. It can happen here, when a read ends inside a character written as several bytes, such as an accented or a Japanese letter. But plain English replies break the parser too: a read that ends 20 bytes into an event, or holds two events, raises a JSON error with no such letter in sight.

</details>

<details><summary>C. Fails partway through some replies with a JSON error</summary>

✓ Right. Some replies stop partway on a `JSONDecodeError`, raised when a read ended inside an event or carried two of them. Now and then an event also vanishes without a trace.

Errors appear on the on-call dashboard at random: a failed prompt succeeds when sent again, and the laptop test passes every time. Customers whose replies get through find a word missing here and there, from the events that vanished quietly.

A connection delivers a run of bytes, and how much each read returns depends on the network and the buffers along the way, not on where an event ends. An event is complete only at the blank line after it. A read that ends 20 bytes into an event leaves an unfinished string, and `json.loads` raises "Unterminated string"; a read that holds two events raises "Extra data"; a read cut inside the `data: ` prefix fails the test, and that event is lost without an error. Hold back whatever follows the last complete line of a read, put it in front of the next read, parse only whole lines, and handle an event at each blank line. Decode the bytes with an incremental decoder too, one that holds an unfinished character until the rest of it arrives: where a server sends a reply in Japanese or Greek as plain characters, a read can end inside one of them, and decoding that read alone raises `UnicodeDecodeError`. This holds for any stream your code reads from a connection, whoever sends it, and for a server that sends each event as a line of JSON, since a line can be split across reads as well.

</details>

<details><summary>D. Drops a word now and then, and never raises an error</summary>

✗ A read that starts in the middle of an event fails the `startswith` test and is skipped without a word, and that does happen. Most cuts fall elsewhere: a read that ends inside an event, or holds two of them, still starts with `data: `, and `json.loads` raises on it.

</details>

---

###### 9. Retrying a recipe stream after a dropped connection

On a cooking site, an assistant streams recipes into a chat bubble. `call_model` yields events, and `show` adds each piece of text to the end of the bubble. When the connection between your server and the provider's API drops, the loop raises `ConnectionError`, and your code tries again with the same request. One evening a network fault drops the connection halfway through a sponge cake recipe.

```python
def stream_recipe(messages):
    for attempt in range(3):
        try:
            for event in call_model(messages=messages, stream=True):
                if event.kind == "text":
                    show(event.text)
            return
        except ConnectionError:
            continue
```

What does the visitor see in the bubble once the retry finishes?

<details><summary>A. The recipe once, the retry picking up where it broke off</summary>

✗ A download resumes where it broke off, and you would hope a retry of the same request does likewise. The model remembers nothing of the first call, and the request says nothing about the half already shown. The second call writes a new reply from its first word, and `show` adds it after the half that is there.

</details>

<details><summary>B. Half the recipe, then the recipe again from its first line</summary>

✓ Right. The bubble shows the first half of the recipe, then a whole recipe from its first line, written by the second call. Nothing removes the half that was already shown.

The visitor reads "Whisk two eggs with sugar. Fold in t" run straight into "Whisk two eggs" again, and the second version can differ from the first in its amounts or its steps. Someone baking from the screen now has a recipe and a half and no sign of which to follow.

A retry sends the same request again, and the model reads each call from scratch, so the second call starts a new reply from its beginning; under the usual settings the model picks each token with some chance in it, so the new reply can be worded differently, too. A stream that broke ended without its end event, and that is the sign to retry. Before the retry starts, clear what the first attempt showed. To keep that text instead, send a new request that carries it and asks the model to continue from there. A server you run yourself answers the repeated request as a new call too, with a new reply.

</details>

<details><summary>C. Half the recipe only, with the retry refused as a repeat</summary>

✗ The second request is word for word the first, which can make it look like a duplicate that a careful server would turn away. A model call has no such guard: the same request sent again is a new call, and the model writes a new reply. It streams into the bubble after the half that stayed.

</details>

---

###### 10. Asking a follow-up after a tutoring stream breaks

A maths tutoring app streams explanations to students and keeps each chat in `history`, which starts with the system prompt (the app's standing instructions to the model) and is the base of every request. A student asks how to solve 2x + 3 = 11 and reads the first step; then the stream breaks with a `ConnectionError`, and the app shows "Connection lost". The student types: "What did you mean by the second step?"

```python
def ask(question):
    reply = ""
    request = history + [{"role": "user", "content": question}]
    for event in call_model(messages=request, stream=True):
        if event.kind == "text":
            reply += event.text
            show(event.text)
    history.append({"role": "user", "content": question})
    history.append({"role": "assistant", "content": reply})
```

What does the request for the new question carry from the broken turn?

<details><summary>A. The lines the student read, with no mark of the break</summary>

✗ The student read those lines on screen, so they feel like part of the chat. The screen and the history are separate: `history` gets its two new messages only after the loop finishes, and the broken stream left `ask` before that. The words the student read were never saved, and the next request does not carry them.

</details>

<details><summary>B. The whole explanation, as the model finished it</summary>

✗ The model may have gone on writing for a moment, and a provider could keep the finished reply for the next call. Yet the model reads only the request your code builds, and that request comes from `history`, which holds nothing of that turn.

</details>

<details><summary>C. The lines the student read, with a note that they were cut off</summary>

✗ The student saw the reply cut off under "Connection lost", and the request could carry the same mark. Nothing in this code writes one, and nothing between your code and the model adds it. The request carries what `history` holds, and that is nothing of the turn that broke.

</details>

<details><summary>D. Neither the first question nor any line of its reply</summary>

✓ Right. The request carries the system prompt and the new question, and nothing of the turn that broke: neither the equation nor the step the student read.

The model is asked what it meant by a second step it never wrote, in a chat where no equation was ever asked about. It can only guess, and a confident answer about some other second step leaves the student more lost than before.

What the student saw and what the next request carries are two separate records, and a stream that breaks writes only to the first. The model keeps nothing between calls, so the request is all it has, and this code adds to `history` only after a stream has ended. Decide what a broken turn leaves behind: save the question and the part shown, with a note that the reply was cut off, so the model can pick up from it; or save nothing and ask the student to send the question again. Check for the end event before you save a reply as whole. Your own server keeps no history between calls either, so this choice falls to your code there too.

</details>

---

## Further reading

- [Orca](https://www.usenix.org/conference/osdi22/presentation/yu), 2022: why a model writes its reply one token per pass, and how a serving system schedules many requests around that step.
- [DistServe](https://arxiv.org/abs/2401.09670), 2024: time to first token and time per output token treated as two separate targets, met by running the reading and the writing on separate GPUs.

Next: [Images, PDFs and audio](../images-pdfs-and-audio/README.md)
