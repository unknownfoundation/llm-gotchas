# Structured outputs

Prev: [Prompting](../prompting/README.md) · Next: [Tool calling](../tool-calling/README.md) · Reviewed October 2026

A structured output is a model reply your code parses into fixed fields, most often a JSON object. The model writes it token by token like any other reply, and the request adds a description of the fields and, in some ways of asking, a limit on which tokens may be drawn.

- **Token**: a piece of text, often a word or part of one; the model writes a reply one token at a time, drawing each at random from its vocabulary, the fixed list of tokens it knows, with higher-scored tokens more likely.
- **JSON Schema**: a standard way to describe JSON: the fields of an object, which of them every object must have (the required fields), the type of each value (string, number, integer, boolean, array, object or null), and limits a value must meet, such as a minimum, a pattern or a fixed list of allowed values, called an enum.
- **Constrained decoding**: generation in which, on each step, every token that would take the text outside a given format is removed from the model's scores before the next token is drawn.
- **Grammar**: rules for which text may come next; a server builds one from your schema for constrained decoding.
- **Tool call**: the part of a reply that asks your code to run a function described in the request, with the arguments as JSON; a request can force every reply to be a call to one named tool.
- **Generation cap**: the request's limit on how many tokens the reply may run to, often called max tokens.
- **Finish reason**: the field in the response that says why the reply ended, such as a natural end or the cap.

On many provider APIs, JSON mode and schema mode are request settings, and each API's schema mode accepts only part of JSON Schema; a server you run yourself turns your schema into a grammar for constrained decoding, and some also take a grammar you write or a list of allowed replies; in a chat app you can ask for a format only in your message, though some coding assistants take a schema when a script runs them and check their final answer against it.

Four ways to ask for structured output are common:

| Way | What it does | Where you find it |
| --- | --- | --- |
| Prompt | describes the fields in the instructions, often with a filled-in example | every kind of system |
| JSON mode | a request setting that holds the reply to JSON | many provider APIs, your own server |
| Schema mode | a request setting that takes your JSON Schema and writes the reply under constrained decoding; some systems also let you turn it on for one tool's arguments | many provider APIs, your own server |
| Tool arguments | a tool whose arguments are the fields you want, and a request that forces a call to it; your code reads the call's arguments | many provider APIs, your own server |

Schema mode goes by other names too; some systems call it strict mode.

The problems show at the parse and after it, in what your code assumes the object holds.

This topic lets you:

- choose how to ask a model for JSON, from a prompt or a forced tool call to schema mode, for a catalogue importer or a homework helper that works step by step
- design a schema and its prompt together, from the order of the fields to what an input may lack
- write the code that parses a streamed or finished reply, checks it and passes it to a database or a web page

###### 1. Asking an importer for three fields

An online shop imports supplier catalogues with a model: your code reads `name`, `price` and `colour` from each of its replies and saves what it finds. The prompt lists the three fields with a filled-in example, yet a few replies a week arrive without `colour`. Every reply ends well under the generation cap. The team weighs three changes: a firmer prompt with a second example, JSON mode, or schema mode with all three fields required.

Which of the three changes makes the replies without `colour` stop?

<details><summary>A. They stop with any one of the three</summary>

✗ A prompt with a good example gets nearly every reply right when you test by hand, and a second example is the usual next step. A prompt is read, not enforced: the replies without `colour` already came from a prompt with an example, and JSON mode holds the reply only to JSON syntax.

</details>

<details><summary>B. They stop with schema mode alone</summary>

✓ Right. With schema mode on, every reply that ends normally carries `name`, `price` and `colour`. With the firmer prompt or JSON mode, a record still reaches the site now and then with no colour.

The three ways hold the reply to different things. A prompt is read, and nothing compares the reply with it. JSON mode keeps the reply to JSON syntax, with whatever keys the model chose. Schema mode takes your JSON Schema with the request and uses constrained decoding: on each step, the tokens that would break the schema are removed before the draw, so the model cannot close the object while a required field is missing.

Put the fields in the schema, and keep the prompt for what each field means.

</details>

<details><summary>C. They stop with JSON mode and with schema mode</summary>

✗ The setting is named for JSON, and your prompt already spells out the JSON you want. JSON mode holds the reply to valid JSON and leaves the keys to the model, and `{"name": "Linen shirt", "price": 39}` is valid JSON with no colour in it.

</details>

<details><summary>D. They keep arriving whichever change is made</summary>

✗ Each token is drawn by chance, and chance leaves every token some odds. Schema mode changes what can be drawn: tokens that would break the schema are removed before each draw, so a reply that ends normally cannot leave out a required field.

</details>

---

###### 2. Getting a moving quote's fields through a forced tool call

A removals company has a quote form that sends each customer's description of their move to a model on a provider's API. Your code already handles tool calls, so rather than ask for a JSON reply you define a tool, `record_move`, whose arguments are `rooms`, `floor` and `move_date`, all required, and force every reply to be a call to it, a setting your model supports. You leave schema mode off for this tool.

What reaches your code from the API?

<details><summary>A. A call whose arguments match the schema, every time</summary>

✗ The schema sits in the tool definition the API receives, and the API holds it with every call. The model reads the definition and writes the arguments as text; with schema mode off, no token is removed for leaving the schema, and nothing checks the call before it reaches you.

</details>

<details><summary>B. An error from the API in place of any call that breaks the schema</summary>

✗ A web framework rejects a form that fails its rules, and the API holds your schema. Without schema mode on the tool, the API returns each call as the model wrote it, as a normal success, and the checking is left to your code.

</details>

<details><summary>C. Now and then, a call whose arguments break the schema</summary>

✓ Right. Every reply is a call to `record_move`, and its arguments usually fit the schema, but nothing makes them: a call can carry `floor` as `"third"` instead of 3, which your code cannot use as a number, or leave `move_date` out.

On a provider's API, forcing the call decides that the reply is a call to that tool and says nothing about how the arguments are written. Tool arguments are text the model writes, like any reply, and the schema in the tool definition is part of what it reads. Without schema mode, nothing removes a token that leaves the schema, so the arguments are the model's attempt at it.

Turn schema mode on for the tool, so the arguments are written under constrained decoding, or check them with a validator and ask again on a failure. APIs differ in the default: on some, schema mode is on for a tool unless you turn it off, as long as the tool's schema is one it accepts; on others it stays off until you turn it on.

</details>

---

###### 3. Adding a privacy category to a support router's prompt

An online bike shop routes support messages under schema mode, and the schema's `category` field is an enum of `billing`, `shipping` and `returns`. A weekly chart counts messages per category. When a new privacy law brings about 40 requests a week to delete customer data, a colleague adds a line to the prompt: use `privacy` for those requests. The schema stays as it was.

Where do the deletion requests show up on the weekly chart?

<details><summary>A. In a new bar, <code>privacy</code></summary>

✗ The prompt line names this exact case and was written after the schema, and the newer instruction usually wins. The two are not weighed against each other. The schema decides which tokens may be drawn in `category`, and `privacy` is not among them, however plainly the prompt asks for it.

</details>

<details><summary>B. Some in a new <code>privacy</code> bar, the rest in the old three</summary>

✗ A firm, specific prompt line usually wins some ground with a model. Schema mode is not weighed against the prompt: once `"category": "` is written, only the three listed values can follow, on every reply.

</details>

<details><summary>C. Inside the three existing bars</summary>

✓ Right. Each deletion request is filed as `billing`, `shipping` or `returns`, and those bars grow by about 40 a week between them while no `privacy` bar appears.

Under schema mode, the schema alone decides which tokens are removed; the prompt is text the model reads. After `"category": "` is written, the tokens that would start any word other than the three listed are removed, so `privacy` cannot be drawn, and the model writes the listed value it scores highest. Keys follow the same rule: when the schema admits only the keys it lists, a key the prompt asks for and the schema leaves out is never written.

Add `privacy` to the enum and keep the prompt line, which tells the model when to use it. A value such as `other`, with its weekly count watched, is how the next new kind of message shows up.

</details>

<details><summary>D. Nowhere; the requests fail with an error</summary>

✗ A prompt that names a value the schema forbids contradicts the schema, and APIs refuse requests they can see are invalid. The request is valid, the prompt is read as text, and each reply comes back with one of the three values.

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

✓ Right. `record["weight_kg"]` holds a number, such as 0 or a plausible weight, though the note shows none. The record parses and passes the schema, so those invented weights sit in your data among the real ones.

The schema makes `weight_kg` required and types it as a number, and constrained decoding enforces both. Once the reply reaches `"weight_kg": `, the only tokens left are those that can start a number; `null`, the closing brace and the next key are all removed. The model writes the best number it can, and on a note without a weight that number is a guess, since the schema has no place for "not on the note".

Let the type include null, as `["number", "null"]`, and say in the field's description or the prompt to use null when the note shows no weight; your code then reads `None` as missing.

</details>

<details><summary>B. A JSON null, read as <code>None</code></summary>

✗ JSON has null for exactly this, and a person filling in a form leaves a blank when the paper has no answer. The schema rules it out: the type is number, and null is not a number. Under schema mode the null token is removed before it can be drawn, so the model has to write digits.

</details>

<details><summary>C. A missing key: reading it raises <code>KeyError</code></summary>

✗ Leaving a field out is what people do when there is nothing to put in it, and replies asked for by prompt alone skip fields. Here the schema lists `weight_kg` as required, and schema mode will not let the reply close until the key and a value are written.

</details>

<details><summary>D. No reply at all: <code>call_model</code> raises an error for that note</summary>

✗ APIs refuse invalid requests, and this note cannot fill a required field. The request is valid, since the schema is well formed and the note is text, and schema mode has no step that compares the note with the schema: it shapes the reply as it is written.

</details>

---

###### 5. Listing the findings first in a code-review bot's schema

A software company runs a code-review bot under schema mode on a provider's API, and it returns `verdict`, either `approve` or `reject`, and `findings`, a list of problems; both are required. The schema lists `verdict` first. A senior engineer asks for `findings` to be listed first in the schema, hoping the verdict will then follow from them. A colleague answers that the keys of a JSON object have no order, so moving the field changes nothing.

Once `findings` is listed first in the schema, which field does each of the bot's replies open with?

<details><summary>A. <code>findings</code></summary>

✓ Right. Every reply now opens with `findings` and closes with `verdict`. Before the change every reply opened with `verdict`, written before a single finding existed.

A program that reads a JSON object finds its keys by name, so their order means nothing to it, and the colleague is right about that. The model, though, writes the reply from front to back, and schema mode builds its constraint from the schema with the keys in sequence: after `{`, the only key that may come is the first one the schema lists, then the next. So the order of the schema's properties is the order of writing. Some systems write required fields before optional ones, so the listed order can change when some fields are optional; both are required here.

List a field that should shape another above it.

</details>

<details><summary>B. <code>verdict</code>, as before</summary>

✗ The colleague is right that a program reading JSON finds keys by name, whatever their order, and JSON itself leaves the order of keys free. Schema mode allows only the key the schema lists next, so with `findings` listed first, `"findings"` is the only key a reply can open with.

</details>

<details><summary>C. Either one, varying from reply to reply</summary>

✗ A JSON object's keys have no order, and a schema lists them without saying which comes first. The constraint is built from the schema with its keys in sequence, so the order is fixed: every reply opens with the field listed first.

</details>

---

###### 6. Saving an invoice whose total the model misread

A bookkeeping app reads scanned supplier invoices through schema mode. The schema requires `supplier`, `due_date`, `lines` with an `amount` on each, and `total`, a number. Before anything reaches the ledger, your code checks the reply against the same schema with a validator. Say the model misreads one smudged total, 1,420.00, as 1,240.00.

What reaches the ledger from that invoice?

<details><summary>A. Nothing; your validator rejects the reply and stops the save</summary>

✗ A check before every save is the safety net you built for bad data, and this total is bad. A validator compares the reply with the schema, which asks only that `total` be a number; the misread total is still a number, so the reply passes, as would a due date of February 30 if the schema typed it as a plain string.

</details>

<details><summary>B. A total of 1,240.00, flagged as not matching the lines</summary>

✗ The schema lists the line amounts and the total side by side, and a person reading them together would spot the gap at once. A schema describes each value on its own and never adds the lines to compare the sum with the total, so the validator stays silent and the invoice goes through unflagged.

</details>

<details><summary>C. A total of 1,240.00, with no flag on it</summary>

✓ Right. The reply passes the validator, and the ledger records a total of 1,240.00 with nothing on the way to flag it, so the ledger holds 180.00 less than the invoice.

A schema describes each value on its own: which fields exist, the type of each, which are required, and any limits such as a minimum or a pattern, which a validator does check. A limit would not have caught this total, a positive number like the real one. Constrained decoding keeps the reply within the schema and the validator checks it once more, and neither compares one field with another or with the scan: a total that does not match the lines, a due date of February 30 in a plain string field and a supplier name that appears nowhere on the page all pass both.

Checks of meaning are your code's job: add up the lines and compare them with the total, parse the due date as a real date, and look the supplier up in the list of suppliers you already pay. If the lines were read right, they add up to 1,420.00, and a sum check would hold this invoice back for a person to look at.

</details>

<details><summary>D. Nothing; the model call fails with an error before any reply</summary>

✗ Schema mode is a setting on the request, and the API enforces it, the way it enforces the generation cap. Schema mode holds the reply to the format while it is written: a reply that fits the format comes back as a normal success, and this one fits, since a misread number is still a number.

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

✗ Every piece arrives under schema mode, which promises a reply that fits the schema. Schema mode holds the reply to the schema token by token, so each piece is the start of a valid reply, and the start of an object is not a whole JSON document: every parse before the closing brace throws, and the card stays blank until the end.

</details>

<details><summary>B. Each field, appearing once its value is complete</summary>

✗ A value broken off halfway is the usual thing that stops a parse, and between two fields no value is broken. Right after the headline's closing quote, the text is still an object with no closing brace, and `JSON.parse` throws on it the same way. Only the last piece closes the object and gives the parser a whole document.

</details>

<details><summary>C. An error from the request, and no stream at all</summary>

✗ A validator needs the whole document before it can check it, and schema mode checks the reply against a schema. Schema mode works token by token, so the reply streams like any other: the pieces arrive as they are written, each one the start of a valid reply.

</details>

<details><summary>D. Nothing until the end, then all three fields at once</summary>

✓ Right. Every parse but the last one throws, so the card stays blank while the model writes and fills in all three fields together when the closing brace arrives: the same wait as with no streaming, and the error log stays empty, since the `catch` swallows every failure.

A JSON object is a whole document only once its closing brace is written. Every shorter piece is an object still open, and a JSON parser refuses it wherever the piece ends: inside a string, after a comma or right after a complete value. Schema mode makes the finished reply fit the schema, and while the reply streams, the text so far is only the start of one. Cut a 127-character reply of this shape after any character, and only the cut after the last one leaves text that parses.

To show fields as they arrive, use a parser made for incomplete JSON, which closes whatever is still open and returns the fields finished so far, or ask for a format your code can read line by line; run the full parse after the stream ends and the finish reason says the reply is complete.

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

✗ Schema mode knows the whole schema in advance, and the cap is set on the same request. Constrained decoding works one token at a time and knows nothing of how many tokens are left: the model writes clauses as if there were no cap, and the cap stops it partway through one.

</details>

<details><summary>B. The start of the object, broken off mid-clause</summary>

✓ Right. `json.loads` gets the start of the object, the policy number and the first clauses, broken off partway through a clause, and raises `JSONDecodeError`, Python's error for text that is not valid JSON. The response arrives as a success with a finish reason that names the cap, and since the parse fails before the loop, not one clause of a long policy is saved.

Constrained decoding chooses which tokens may come next from the text written so far, and the number of tokens left plays no part in that choice, so nothing steers the reply toward an early close. Generation halts when the count reaches the cap, with the clause, the list and the object still open, and the promise of schema mode covers only a reply that ended normally.

Read the finish reason before you parse; on a cut, raise the cap, or split the policy and extract its clauses in parts. A retry under the same cap is cut again.

</details>

<details><summary>C. The whole object, written past the cap</summary>

✗ You turned schema mode on so that replies always fit the schema, and a reply cut short does not fit. The cap halts generation at its 1,000 tokens, whatever the format still needs, so the object for a policy that needs several thousand stops mid-clause and `json.loads` raises on it.

</details>

<details><summary>D. Empty text, with a finish reason that names the cap</summary>

✗ Every reply you have seen under schema mode fitted the schema, and a cut reply cannot. The response carries the text written up to the cap, along with a finish reason that names the cap, and that cut text is what reaches `json.loads`.

</details>

---

###### 9. Asking a homework helper for its answer as JSON

A homework app explains maths word problems step by step. It runs on a chat model that writes its reply straight away, not a reasoning model that thinks before it answers, and on your eval set it gets most final answers right in free text. The mobile team now wants each reply as JSON, with a `steps` field first and an `answer` field after it. In the one-call version, the model writes its steps and answer as that JSON directly. In the two-call version, the free-text call stays, and a second call copies its steps and answer into the JSON.

When you run both versions on the eval set, what scores should you expect against free text?

<details><summary>A. Both versions about as well as free text</summary>

✗ The steps still come before the answer, so the model reasons first either way, as the usual advice for reasoning in JSON asks. In a 2024 study, prompts that asked for JSON with the reasoning first still scored below free text on reasoning tasks in most of its comparisons, so the one-call version falls below free text, and only the two-call version keeps level with it.

</details>

<details><summary>B. The one-call version as well as free text, the two-call version lower</summary>

✗ One call has fewer moving parts, and a second call that copies text is one more place for a slip. A 2024 study found one small model losing a little to conversion errors, but for most models the converted answers scored nearly the same as free text; the loss came from answering inside the format, so it is the one-call version that drops, and the two-call version stays close to free text.

</details>

<details><summary>C. The two-call version close to free text, the one-call version lower</summary>

✓ Right. The two-call version should score close to free text, and the one-call JSON version lower. A 2024 study, Let Me Speak Freely?, found that the models of that time reasoned less accurately when they had to answer inside a format such as JSON, while free text converted by a second step scored nearly the same as free text for most of them.

The format is part of what each token is drawn from, so asking for JSON in the reasoning call can change the reasoning itself. The study ran the same tasks four ways: free text; JSON asked for in the prompt, reasoning field first; JSON held by constrained decoding; and free text that a second call converted to JSON. On reasoning tasks free text did best in most cases, the converted version close behind and the formats lower; on classification tasks, which pick one label from a fixed list, strict JSON did about as well, sometimes better.

Keep the reasoning in free text and use schema mode for the copy, where the format costs little, and run both versions on your own eval set before you choose.

</details>

<details><summary>D. The one-call version better than free text</summary>

✗ A `steps` field makes the model show its work in order, and showing work is what a good tutor asks of a pupil. A 2024 study found the opposite on reasoning tasks: JSON with the reasoning first scored below free text in most comparisons, and free text converted afterwards came closest to it, so the steps field costs the one-call version accuracy instead of adding any.

</details>

---

###### 10. Showing 19-digit order numbers on a dashboard

A furniture shop has a returns tool that reads each customer email through schema mode, and the schema types `order_number` as an integer. Your Python service parses the reply, logs it and passes the same JSON on to a dashboard in the warehouse staff's browser. Order numbers are 19-digit ids made by the order system. Today's email is about order 1846029371564823719, and the service's log shows exactly that number.

What does the dashboard show as the order number?

<details><summary>A. The same number as the log</summary>

✗ Your log shows the right number, the dashboard reads the same JSON, and JSON carries the digits as written. The two programs read numbers differently: Python keeps an integer exact at any size, while the browser's parser turns it into a 64-bit floating-point value, which cannot hold this 19-digit id exactly.

</details>

<details><summary>B. A negative number</summary>

✗ A fixed-width integer wraps round to a negative value when it gets too big, and a 19-digit id is big enough to make that worry real. The browser's parser reads every number as a 64-bit float, which has a sign of its own and loses exactness in the low digits instead of wrapping. The id stays positive and keeps its 19 digits.

</details>

<details><summary>C. A shorter number, last digits dropped</summary>

✗ A narrow table column cuts a long value to fit, and a 19-digit id is the kind that gets cut. The parser keeps every digit's place, so the number on screen is as long as the one in the log. Only its last digits differ, because it has been rounded to the nearest value a 64-bit float can hold.

</details>

<details><summary>D. A slightly different number</summary>

✓ Right. The dashboard shows 1846029371564823800, which differs from the log in its last digits. Nothing fails, and the wrong id has the shape of any other, so a search for it finds a different order or none.

JSON writes a number as digits and leaves its precision to the program that reads it. Python reads an integer into an exact whole number of any size, which is why the log is right. JavaScript in the browser, like many JSON readers, turns every number into a 64-bit floating-point value, and such values hold every whole number exactly only up to 2^53, 9007199254740992, a 16-digit number. At this id's size a 64-bit float holds only every 256th whole number, and an odd id such as this one is never among them, so the browser keeps the nearest one it can hold, 1846029371564823808, and prints it as 1846029371564823800. The JSON standard warns that integers outside that range may not read the same everywhere.

Type identifiers as strings in the JSON Schema, digits and all, so schema mode makes the model write them in quotes and they arrive in the browser digit for digit; nobody does arithmetic on an order number.

</details>

---

## Further reading

- [Let Me Speak Freely?](https://arxiv.org/abs/2408.02442), 2024: reasoning accuracy declined when models had to answer in a fixed format such as JSON, while classification accuracy held or rose.
- [Grammar-Constrained Decoding for Structured NLP Tasks without Finetuning](https://arxiv.org/abs/2305.13971), 2023: a grammar written for each task, even one that depends on the input, keeps an off-the-shelf model's output to valid structures, and in its tests beat unconstrained models on extraction and parsing.

Prev: [Prompting](../prompting/README.md) · Next: [Tool calling](../tool-calling/README.md)
