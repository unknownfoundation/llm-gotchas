# Structured outputs

Needs: [Generation parameters](../generation-parameters/README.md) · [Prompting](../prompting/README.md) · Reviewed October 2026

A structured output is a model reply your code can parse into fixed fields, most often a JSON object. The model writes it like any other reply, one token at a time: on each step it scores every token in its vocabulary, and one is drawn at random, the higher-scored ones more likely. The request adds a description of the fields and, in some ways of asking, a limit on which tokens may be drawn.

- **Token**: a piece of text, often a word or part of one; the model writes a reply one token at a time, each drawn from the fixed list of tokens it knows, its vocabulary.
- **JSON Schema**: a standard way to describe JSON: the fields of an object, which of them every object must have (the required fields), the type of each value (string, number, integer, boolean, array, object or null), and limits a value must meet, such as a minimum, a pattern or a fixed list of allowed values, called an enum.
- **Constrained decoding**: generation in which, on each step, every token that would take the text outside a given format is removed before the next token is drawn. It works on the model's scores, so a model that follows instructions poorly keeps the format as surely as one that follows them well.
- **Grammar**: rules for which text may come next; a server builds one from your schema for constrained decoding.
- **Tool call**: the part of a reply that asks your code to run a function described in the request, with the arguments as JSON; a request can force every reply to be a call to one named tool.
- **Generation cap**: the request's limit on how many tokens the reply may run to, often called max tokens.
- **Finish reason**: the field in the response that says why the reply ended, such as a natural end or the cap.

How much a system lets you ask for beyond a line in the prompt:

- **A provider's API**: JSON mode and schema mode are request settings on many of them, and each API's schema mode accepts only part of JSON Schema, with limits of its own.
- **Your own server**: the server turns your schema into a grammar for constrained decoding, and some also take a grammar you write yourself or a list of allowed replies.
- **A ready-made app**: in a chat app you can ask for a format only in your message. Some coding assistants take a schema when a script runs them, and check their final answer against it.

Constrained decoding works the same wherever it runs.

Four ways to ask for structured output are common:

| Way | What it does | Where you find it |
| --- | --- | --- |
| Prompt | describes the fields in the instructions, often with a filled-in example | every kind of system |
| JSON mode | a request setting that holds the reply to JSON | many provider APIs, your own server |
| Schema mode | a request setting that takes your JSON Schema and writes the reply under constrained decoding; some systems also let you turn it on for one tool's arguments | many provider APIs, your own server |
| Tool arguments | a tool whose arguments are the fields you want, and a request that forces a call to it; your code reads the call's arguments | many provider APIs, your own server |

Schema mode goes by other names too; some systems call it strict mode.

Structured output bites at the parse and after it: in what your code assumes the object holds, and in what happens to it on its way into a database or a browser.

This topic lets you:

- choose how to ask a model for JSON, from a prompt or a forced tool call to schema mode, for a catalogue importer or a homework helper that works step by step
- design a schema and its prompt together, from the order of the fields to what an input may lack
- write the code that reads the reply and checks it, from the first streamed piece to the long id a browser shows

###### 1. Asking an importer for three fields

An online shop imports supplier catalogues with a model: your code reads `name`, `price` and `colour` from each of its replies and saves what it finds. The prompt lists the three fields with a filled-in example, yet a few replies a week arrive without `colour`. Every reply ends well under the generation cap. The team weighs three changes: a firmer prompt with a second example, JSON mode, or schema mode with all three fields required.

Which of the three changes makes the replies without `colour` stop?

<details><summary>A. They stop with any one of the three</summary>

✗ A prompt with a good example gets nearly every reply right when you test by hand, and a second example is the usual next step. The prompt is text the model reads, and nothing in the request holds the reply to it: the replies without `colour` already came from a prompt with an example. JSON mode, for its part, holds the reply only to JSON syntax.

</details>

<details><summary>B. They stop with schema mode alone</summary>

✓ Right. With schema mode on, every reply that ends normally carries `name`, `price` and `colour`. The firmer prompt and JSON mode leave the gap open.

With either of the other two, a shirt still reaches the shop's site now and then with no colour, and a shopper who filters for blue never sees it until someone in merchandising fixes the row by hand.

The three ways hold the reply to different things. A prompt is read, and nothing compares the reply with it. JSON mode keeps the reply to JSON syntax, with whatever keys the model chose. Schema mode takes your JSON Schema with the request and uses constrained decoding: on each step, the tokens that would break the schema are removed before the draw, so the model cannot close the object while a required field is missing. Put the fields in the schema, and keep the prompt for what each field means. In a chat app only the prompt is open to you, so check each reply in your code.

</details>

<details><summary>C. They stop with JSON mode and with schema mode</summary>

✗ The setting is named for JSON, and your prompt already spells out the JSON you want, so the two should meet. JSON mode holds the reply to valid JSON and leaves the keys to the model, and `{"name": "Linen shirt", "price": 39}` is valid JSON with no colour in it.

</details>

<details><summary>D. They keep arriving whichever change is made</summary>

✗ Each token is drawn by chance, and it is hard to see how a setting could rule anything out. Schema mode changes what can be drawn: tokens that would break the schema are removed before each draw, so a reply that ends normally cannot leave out a required field.

</details>

---

###### 2. Getting a moving quote's fields through a forced tool call

A removals company has a quote form that sends each customer's description of their move to a model on a provider's API. Your code already handles tool calls, so rather than ask for a JSON reply you define a tool, `record_move`, whose arguments are `rooms`, `floor` and `move_date`, all required, and force every reply to be a call to it, a setting your model supports. You leave schema mode off for this tool.

What reaches your code from the API?

<details><summary>A. A call whose arguments match the schema, every time</summary>

✗ The schema sits in the tool definition the API receives, so it is natural to take it as a contract the API keeps. The model reads the definition and writes the arguments as text, one token at a time. With schema mode off for the tool, no token is removed for leaving the schema, and nothing checks the call before it reaches you.

</details>

<details><summary>B. An error from the API in place of any call that breaks the schema</summary>

✗ A web framework rejects a form that fails its rules, and an API that holds your schema could do the same. Without schema mode on the tool, the API returns each call as the model wrote it, as a normal success, and the checking is left to your code.

</details>

<details><summary>C. Now and then, a call whose arguments break the schema</summary>

✓ Right. Every reply is a call to `record_move`, and its arguments usually fit the schema, but nothing makes them: a call can carry `floor` as `"third"` instead of 3, or leave `move_date` out.

The quote engine multiplies the price by the floor, so the call with `"third"` raises an exception and the quote form answers with an error page. A family pricing a move out of a third-floor flat gets no quote, gives up and books another company's van that evening.

Forcing the call decides that the reply is a call to that tool; it says nothing about how the arguments are written. Tool arguments are text the model writes, like any reply, and the schema in the tool definition is part of what it reads. Without schema mode, nothing removes a token that leaves the schema, so the arguments are the model's attempt at it. Turn schema mode on for the tool, and the arguments are written under constrained decoding, as a reply is under schema mode; or check the arguments with a validator and ask again on a failure. APIs differ in the default. On some, schema mode is on for a tool unless you turn it off, as long as the tool's schema is one it accepts; on others it stays off until you turn it on. A server you run yourself may write a call forced to one named tool under constrained decoding anyway, so its arguments fit the schema even with schema mode off.

</details>

---

###### 3. Adding a privacy category to a support router's prompt

An online bike shop routes support messages under schema mode, and the schema's `category` field is an enum of `billing`, `shipping` and `returns`. A weekly chart counts messages per category. When a new privacy law brings about 40 requests a week to delete customer data, a colleague adds a line to the prompt: use `privacy` for those requests. The schema stays as it was.

Where do the deletion requests show up on the weekly chart?

<details><summary>A. In a new bar, <code>privacy</code></summary>

✗ The prompt line names this exact case, and it was written after the schema, so it reads as the rule now in force. The two are not weighed against each other. The schema decides which tokens may be drawn in `category`, and `privacy` is not among them, however plainly the prompt asks for it.

</details>

<details><summary>B. Some in a new <code>privacy</code> bar, the rest in the old three</summary>

✗ If schema mode were one more instruction, a firm prompt line could pull part of the replies its way. Schema mode is not weighed against the prompt: once `"category": "` is written, only the three listed values can follow, on every reply.

</details>

<details><summary>C. Inside the three existing bars</summary>

✓ Right. Each deletion request is filed as `billing`, `shipping` or `returns`, and those bars grow by about 40 a week between them, while no `privacy` bar ever appears.

The colleague who added the line sees the chart unchanged and concludes the law has brought no requests yet. Meanwhile the requests sit in queues whose agents handle refunds and parcels, and each customer who asked to be deleted gets an answer about a return or a delivery.

Under schema mode, the schema alone decides which tokens are removed; the prompt is text the model reads. After `"category": "` is written, the tokens that would start any word other than the three listed are removed, so `privacy` cannot be drawn. The model writes the listed value it scores highest, and nothing in the reply marks the misfit. Add `privacy` to the enum and keep the prompt line, which tells the model when to use it; a value such as `other`, with its weekly count watched, is how the next new kind of message shows up. A field works the same way: when the schema admits only the fields it lists, a field the prompt asks for and the schema leaves out is never written.

</details>

<details><summary>D. Nowhere; the requests fail with an error</summary>

✗ A prompt that names a value the schema forbids reads as a request that contradicts itself, and a strict API might refuse one. Nothing compares the prompt with the schema. The request is valid, the prompt is read as text, and each reply comes back with one of the three values.

</details>

---

###### 4. Reading delivery notes that show no weight

A haulage firm turns scanned delivery notes into records with an app that uses schema mode. `call_model` is a helper in your code that calls the model's API: it takes the messages and a schema, and returns the reply text. Some older delivery notes have no weight printed anywhere on them.

```python
DELIVERY_SCHEMA = {
    "type": "object",
    "properties": {
        "delivery_number": {"type": "string"},
        "weight_kg": {"type": "number"},
        "signed_by": {"type": "string"},
    },
    "required": ["delivery_number", "weight_kg", "signed_by"],
    "additionalProperties": False,
}

reply = call_model(messages=[{"role": "user", "content": note_text}], schema=DELIVERY_SCHEMA)
record = json.loads(reply)
```

What does `record["weight_kg"]` hold for one of those notes?

<details><summary>A. A number, though the note shows no weight</summary>

✓ Right. `record["weight_kg"]` holds a number, such as 0 or a plausible weight, though the note shows none. The record parses, passes the schema and sits in the table like every other.

At the end of the month the dispatcher's tonnage report adds those invented weights to the real ones, and a pallet recorded at 0 kg goes onto a lorry whose load nobody planned for.

The schema makes `weight_kg` required and types it as a number, and constrained decoding enforces both. Once the reply reaches `"weight_kg": `, the only tokens left are those that can start a number; `null`, the closing brace and the next key are all removed. The model writes the best number it can, and on a note without a weight that number is a guess. Nothing in this schema has a place for "not on the note", so the model is made to fill the gap. Let the type include null, as `["number", "null"]`, and say in the field's description or the prompt to use null when the note shows no weight; your code then reads `None` as missing.

</details>

<details><summary>B. A JSON null, read as <code>None</code></summary>

✗ JSON has null for exactly this, and a person filling in a form leaves a blank when the paper has no answer. The schema rules it out: the type is number, and null is not a number. Under schema mode the null token is removed before it can be drawn, so the model has to write digits.

</details>

<details><summary>C. A missing key: reading it raises <code>KeyError</code></summary>

✗ Leaving a field out is the obvious move when there is nothing to put in it, and replies asked for by prompt alone do skip fields. Here the schema lists `weight_kg` as required, and schema mode will not let the reply close until the key and a value are written, so the key is always there with a value in it.

</details>

<details><summary>D. No reply at all: <code>call_model</code> raises an error for that note</summary>

✗ An input that cannot fill a required field sounds like an invalid request, the kind an API refuses. The request is valid: the schema is well formed, and the note is text. Schema mode has no step that compares the note with the schema. It shapes the reply as it is written, and the reply it shapes holds a number.

</details>

---

###### 5. Listing the findings first in a code-review bot's schema

A software company runs a code-review bot under schema mode on a provider's API, and it returns `verdict`, either `approve` or `reject`, and `findings`, a list of problems; both are required. The schema lists `verdict` first. A senior engineer asks for `findings` to be listed first in the schema, hoping the verdict will then follow from them. A colleague answers that the keys of a JSON object have no order, so moving the field changes nothing.

Once `findings` is listed first in the schema, which field does each of the bot's replies open with?

<details><summary>A. <code>findings</code></summary>

✓ Right. Every reply now opens with `findings` and closes with `verdict`, where before the change every reply opened with `verdict`.

Leave the schema as it was on the colleague's word, and each verdict keeps being written before a single finding exists. A developer whose pull request was rejected reads findings written after the reject and in its light, and a problem the model noticed only while listing them could never have turned an approve into a reject.

A program that reads a JSON object finds its keys by name, so their order means nothing to it. The model, though, writes the reply from front to back. Schema mode builds its constraint from the schema with the keys in sequence: after `{`, the only key that may come is the first one the schema lists, then the next. So the order of the schema's properties is the order of writing: list a field that should shape another above it, as reasons go above a score. Some systems write required fields before optional ones, so the listed order can change when some fields are optional; both are required here.

</details>

<details><summary>B. <code>verdict</code>, as before</summary>

✗ The colleague is right that a program reading JSON finds keys by name, whatever their order, and from there it is a short step to thinking the model's habit decides. The model writes the reply front to back, and schema mode allows only the key the schema lists next: with `findings` listed first, `"findings"` is the only key a reply can open with.

</details>

<details><summary>C. Either one, varying from reply to reply</summary>

✗ Since a JSON object's keys have no order, a schema that lists both appears to leave the order to the model on each reply. The constraint is built from the schema with its keys in sequence, so the order is fixed: every reply opens with the field listed first.

</details>

---

###### 6. Saving an invoice whose total the model misread

A bookkeeping app reads scanned supplier invoices through schema mode. The schema requires `supplier`, `due_date`, `lines` with an `amount` on each, and `total`, a number. Before anything reaches the ledger, your code checks the reply against the same schema with a validator. Say the model misreads one smudged total, 1,420.00, as 1,240.00.

What reaches the ledger from that invoice?

<details><summary>A. Nothing; your validator rejects the reply and stops the save</summary>

✗ A check that runs before every save is the safety net you built for bad data, and this total is bad. A validator compares the reply with the schema, and the schema asks only that `total` be a number. The misread total is still a number, so the reply passes with no errors, as would a due date of February 30 in a string field.

</details>

<details><summary>B. A total of 1,240.00, flagged as not matching the lines</summary>

✗ The schema lists each line's amount and the total side by side, and a person reading them together would spot the gap at once. A schema describes each value on its own: the total must be a number, and each line must have an amount. Nothing in it adds the lines and compares the sum with the total, so the validator stays silent and nothing flags the invoice.

</details>

<details><summary>C. A total of 1,240.00, with no flag on it</summary>

✓ Right. The reply passes the validator, and the ledger records a total of 1,240.00, with nothing on the way to flag it.

The supplier is paid 180.00 short, and the first anyone hears of it is a reminder letter weeks later, which a bookkeeper has to trace back to one smudged scan.

A schema describes each value on its own: which fields exist, the type of each, which are required, and any limits a value must meet, such as a minimum or a pattern, which a validator does check. A limit would not have caught this total, which is a positive number like the real one. Constrained decoding keeps the reply within the schema, the validator checks the parsed object against the schema once more, and neither compares one field with another or with the scan. A total that does not match the lines, a due date of February 30 in a string field and a supplier name that appears nowhere on the page all pass both. Checks of meaning are your code's job: add up the lines and compare them with the total, parse the due date as a real date, and look the supplier up in the list of suppliers you already pay. If the lines were read right, they add up to 1,420.00, and a sum check would have held this invoice back for a person to look at.

</details>

<details><summary>D. Nothing; the model call fails with an error before any reply</summary>

✗ A setting on the request passes readily for a gate that turns bad replies away. What the setting does is hold the reply to the format while it is written. A reply that fits the format comes back as a normal success, and this one fits, since a misread number is still a number.

</details>

---

###### 7. Showing a listing's fields while the reply streams in

A property site writes listing descriptions with a model under schema mode, as JSON with `headline`, `summary` and `features`. To show the fields as they arrive, the front end streams the reply: `streamReply`, a helper in your code, sends the request and yields the reply's text piece by piece. After each piece, the code tries to parse everything received so far.

```javascript
let received = "";
for await (const piece of streamReply(listingRequest)) {
  received += piece;
  try {
    render(JSON.parse(received));
  } catch {
    continue;
  }
}
```

What does the page show while the reply streams in?

<details><summary>A. Each field, filling in word by word as it arrives</summary>

✗ Schema mode keeps the reply valid, and it is tempting to read that as valid at every moment. It holds the reply to the schema token by token, so each piece is the start of a valid reply, and the start of an object is not a whole JSON document. Every parse before the closing brace throws, and `render` never runs until then.

</details>

<details><summary>B. Each field, appearing once its value is complete</summary>

✗ A value broken off halfway is the obvious thing that stops a parse, so the moments between two fields look safe. Right after the headline's closing quote, the text is still an object with no closing brace, and `JSON.parse` throws on it the same way. Only the last piece closes the object and gives the parser a whole document.

</details>

<details><summary>C. An error from the request, and no stream at all</summary>

✗ A reply held to a schema sounds like something that has to be finished before it can be sent. Schema mode works token by token, so the reply streams like any other: the pieces arrive as they are written, each one the start of a valid reply.

</details>

<details><summary>D. Nothing until the end, then all three fields at once</summary>

✓ Right. Every parse but the last one throws, so the card stays blank while the model writes and then fills in all three fields together when the closing brace arrives.

On a slow connection a house hunter stares at an empty card for several seconds and then sees it fill at once, the same wait as with no streaming. The team that built the streaming path sees no gain on its latency chart and nothing in its error log, since the `catch` swallows every failure.

A JSON object is a whole document only once its closing brace is written. Every shorter piece of it is an object still open, and a JSON parser refuses it wherever the piece ends: inside a string, after a comma or right after a complete value. Schema mode makes the finished reply fit the schema, and while the reply streams, the text so far is only the start of one. Cut this 127-character reply after any character, and only the cut after the last one leaves text that parses. To show fields as they arrive, use a parser made for incomplete JSON, which closes whatever is still open and returns the fields finished so far, or ask for a format your code can read line by line; run the full parse after the stream ends and the finish reason says the reply is complete.

</details>

---

###### 8. Extracting every clause from a long policy

An insurance broker lets customers compare policies clause by clause, and you build the step that turns each policy document into JSON through schema mode. `call_model` is a helper in your code that calls the model's API: it takes the messages, a schema and a generation cap as `token_cap`, and returns the response, with the text the model wrote in `reply.text`. The first 60-page commercial policies arrive next week; assume the full JSON for one runs to several thousand tokens.

```python
import json

def extract_clauses(policy_text):
    reply = call_model(
        messages=[{"role": "user", "content": policy_text}],
        schema=POLICY_SCHEMA,
        token_cap=1000,
    )
    policy = json.loads(reply.text)
    for clause in policy["clauses"]:
        save_clause(policy["policy_number"], clause)
```

What reaches `json.loads` for a 60-page policy?

<details><summary>A. A closed object with only the first clauses in it</summary>

✗ Only a closed object fits the schema, so a mode that promises the schema would have to finish one in time. Constrained decoding works one token at a time: it keeps the reply inside the format and knows nothing of how many tokens are left. The model writes clauses as if there were no cap, and the cap stops it partway through one.

</details>

<details><summary>B. The start of the object, broken off mid-clause</summary>

✓ Right. `json.loads` gets the start of the object, the policy number and the first clauses, broken off partway through a clause, and it raises `JSONDecodeError`, Python's error for text that is not valid JSON. The response arrives as a success, and its finish reason says the generation cap was reached.

The parse fails before the loop, so not one clause of a long policy is saved, not even those finished before the cut. A retry under the same cap is cut again, and on launch morning the broker's account manager finds the commercial policies missing from the comparison page.

Constrained decoding chooses which tokens may come next from the text written so far; the number of tokens left plays no part in that choice, so nothing steers the reply toward an early close. Generation halts when the count reaches the cap, with the clause, the list and the object still open, and the promise of schema mode covers only a reply that ended normally. Read the finish reason before you parse; on a cut, raise the cap, or split the policy and extract its clauses in parts.

</details>

<details><summary>C. The whole object, written past the cap</summary>

✗ A shape guarantee that gave way to a token count would hardly be a guarantee, and the schema is the promise you turned schema mode on for. The cap counts the reply's tokens and halts generation when the count is reached, whatever the format still needs, and a policy that needs several thousand tokens gets only as many as the cap allows.

</details>

<details><summary>D. Empty text, with a finish reason that names the cap</summary>

✗ A reply cut mid-clause cannot match the schema, and holding it back would keep the promise that every reply does. The response carries the text written up to the cap, along with a finish reason that names the cap, and that cut text is what reaches `json.loads`.

</details>

---

###### 9. Asking a homework helper for its answer as JSON

A homework app explains maths word problems step by step. It runs on a chat model that writes its reply straight away, not a reasoning model that thinks before it answers, and on your eval set it gets most final answers right in free text. The mobile team now wants each reply as JSON, with a `steps` field first and an `answer` field after it. In the one-call version, the model writes its steps and answer as that JSON directly. In the two-call version, the free-text call stays, and a second call copies its steps and answer into the JSON.

When you run both versions on the eval set, what scores should you expect against free text?

<details><summary>A. Both versions about as well as free text</summary>

✗ The steps still come before the answer, so the model reasons first either way, as the usual advice for reasoning in JSON asks. That order helps, yet in a 2024 study, prompts that asked for JSON with the reasoning first still scored below free text on reasoning tasks in most of its comparisons.

</details>

<details><summary>B. The one-call version as well as free text, the two-call version lower</summary>

✗ One call has fewer moving parts, and a second call that copies text is one more place for a slip. Copying is not where the study found the loss. One small model did lose a little to conversion errors, but for most models the converted answers scored nearly the same as free text. The loss came from answering inside the format.

</details>

<details><summary>C. The two-call version close to free text, the one-call version lower</summary>

✓ Right. The two-call version should score close to free text, and the one-call JSON version lower. A 2024 study, Let Me Speak Freely?, found models' reasoning accuracy falling when they had to answer inside a format such as JSON. Answering in free text and converting in a second step scored nearly the same as free text for most models.

Ship the one-call version instead, and more pupils than before find a wrong final answer under steps that read well; the parent who rechecks one of them blames the app.

The format is part of what each token is drawn from, so asking for JSON in the reasoning call can change the reasoning, not only its wrapping. The study ran the same tasks four ways: plain free text; JSON asked for in the prompt, reasoning field first; JSON held by constrained decoding; and free text that a second call converted to JSON. On reasoning tasks, free text did best in most cases, the converted version close behind, and the formats lower. On classification tasks, which pick one label from a fixed list, strict JSON did about as well, sometimes better. Keep the reasoning in free text and use schema mode for the copy, where the format costs little. Those were 2024 models, so run both versions on your own eval set before you choose.

</details>

<details><summary>D. The one-call version better than free text</summary>

✗ A `steps` field makes the model show its work in order, and showing work is what a good tutor asks of a pupil. The study found the opposite on reasoning tasks: JSON with the reasoning field first scored below free text in most of its comparisons, and the version that let the model reason in free text came closest to it.

</details>

---

###### 10. Showing 19-digit order numbers on a dashboard

A furniture shop has a returns tool that reads each customer email through schema mode, and the schema types `order_number` as an integer. Your Python service parses the reply, logs it and passes the same JSON on to a dashboard in the warehouse staff's browser. Order numbers are 19-digit ids made by the order system. Today's email is about order 1846029371564823719, and the service's log shows exactly that number.

What does the dashboard show as the order number?

<details><summary>A. The same number as the log</summary>

✗ Your log shows the right number, and the dashboard reads the same JSON, so the same digits are a fair guess. The two programs read numbers differently: Python keeps an integer exact at any size, while the browser's parser turns it into a 64-bit floating-point value, which cannot hold this 19-digit id exactly.

</details>

<details><summary>B. A negative number</summary>

✗ A fixed-width integer wraps round to a negative value when it gets too big, and a 19-digit id sounds big enough to wrap. The browser's parser reads every number as a 64-bit float, which has a sign of its own and loses exactness in the low digits instead of wrapping. The id stays positive and keeps its 19 digits.

</details>

<details><summary>C. A shorter number, last digits dropped</summary>

✗ A narrow table column cuts a long value to fit, and a 19-digit id is the kind that gets cut. The parser keeps every digit's place, so the number on screen is as long as the one in the log. Only its last digits differ, because it has been rounded to the nearest value a 64-bit float can hold.

</details>

<details><summary>D. A slightly different number</summary>

✓ Right. The dashboard shows 1846029371564823800, which differs from the log in its last digits. Nothing fails, and the number on screen has the shape of any other order number.

A packer types the dashboard's number into the order system and pulls up another customer's order, or none at all, and the returned sofa is booked against the wrong one.

JSON writes a number as digits and leaves its precision to the program that reads it. Python reads an integer into an exact whole number of any size, which is why the log is right. JavaScript in the browser, like many JSON readers, turns every number into a 64-bit floating-point value, and those hold every whole number exactly only up to 2^53, which is 9007199254740992, a 16-digit number. This id is far above that: at its size a 64-bit float holds only every 256th whole number, and an odd id such as this one is never among them, so the browser keeps the nearest one it can hold, 1846029371564823808, and prints it as 1846029371564823800. The JSON standard warns that integers outside that range may not read the same everywhere. Type identifiers as strings in the JSON Schema, digits and all, and schema mode makes the model write them in quotes; a string arrives in the browser digit for digit, and nobody does arithmetic on an order number.

</details>

---

## Further reading

- [Let Me Speak Freely?](https://arxiv.org/abs/2408.02442), 2024: reasoning accuracy declined when models had to answer in a fixed format such as JSON, while classification accuracy held or rose.
- [Grammar-Constrained Decoding for Structured NLP Tasks without Finetuning](https://arxiv.org/abs/2305.13971), 2023: a grammar written for each task, even one that depends on the input, keeps an off-the-shelf model's output to valid structures, and in its tests beat unconstrained models on extraction and parsing.

Next: [Tool calling](../tool-calling/README.md)
