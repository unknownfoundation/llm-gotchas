# Generation parameters

Prev: [Inside the model](../inside-the-model/README.md) · Next: [Prompting](../prompting/README.md) · Reviewed October 2026

A model writes its reply one token at a time: on each step it scores every token it knows, and a sampler draws one by those scores. **Generation parameters** are the request fields that act on that draw or end the reply; the ones that act on the draw itself are the **sampling settings**.

- **Token**: a piece of text, often a word or part of one, that the model reads and writes; the tokenizer gives each one a number, its token ID.
- **Logits**: the scores the model gives every token it knows on one step, where a higher score means a likelier next token; a function called softmax turns them into probabilities that add up to 1.
- **Sampler**: the part that draws the next token by those probabilities, so a likely token wins often and a less likely one now and then.
- **Greedy decoding**: taking the single likeliest token on every step instead of drawing; a temperature of 0, a setting in the table below, asks for it.
- **Generation cap**: the request's limit on how many tokens the reply may run to, often called max tokens.
- **Stop sequence**: a string you list in the request that ends the reply when the model generates it.
- **Finish reason**: the field in the response that says why the reply ended, such as a natural end, a stop sequence or the cap.
- **Reasoning model**: a model that writes step-by-step reasoning as tokens before its final answer; your code gets all of that reasoning, a summary or none.

On a provider's API these settings are request fields whose names, defaults and ranges differ by provider and by model, and reasoning models often restrict them; a server you run yourself sets its own defaults and adds settings many APIs lack; a ready-made app picks them for you and shows few of them or none.

The sampling settings; top-k, top-p and min-p are cuts, dropping tokens before the draw:

| Setting | What it does | Where you find it |
| --- | --- | --- |
| Temperature | sharpens or flattens the probabilities: below 1 the likeliest tokens win more often, above 1 rarer ones get picked more | many provider APIs, your own server |
| Top-k | drops all but the k likeliest tokens before the draw | some provider APIs, your own server |
| Top-p, or nucleus sampling | drops the unlikely tail, keeping the fewest likeliest tokens whose probabilities add up to p | many provider APIs, your own server |
| Min-p | drops tokens far less likely than the top one, below a set fraction of its probability | mainly your own server |
| Frequency penalty | makes a token less likely each time it has already appeared in the reply | some provider APIs, your own server |
| Presence penalty | makes a token less likely once it has appeared in the reply at all | some provider APIs, your own server |
| Repetition penalty | makes tokens already in the prompt or the reply less likely | mainly your own server |
| Logit bias | adds a fixed amount, up or down, to the logits of the token IDs you list; at the extreme it bans a token or forces it | some provider APIs, your own server |

These settings usually sit in a shared config or a wrapper function, set once, and their effects surface weeks later.

This topic lets you:

- choose a sampling setting for replies that come out wrong, samey, repetitive or with a word you want gone, and judge before you ship whether it helps
- work out why a reply ended where it did, after a stop string, the generation cap or a reasoning model, and what your code should do with it
- decide what a nightly test can rely on when you need the same reply twice, and what a line typed into a chat app can change

###### 1. Running regression tests at temperature 0

You run nightly regression tests for an app that writes product descriptions, through a provider's API. The model you call applies the temperature you send, and you set it to 0 so each reply can be compared with saved text. Every night the suite sends the same request to the same model twice.

What should you expect from the two calls on a given night?

<details><summary>A. The same text from both calls</summary>

✗ Greedy decoding has no dice in it: it always takes the likeliest token, and both calls send the same input. The rule is fixed, but the logits it reads are computed again on each call and can shift by a hair, enough to flip a near tie and send the rest of the reply down another path.

</details>

<details><summary>B. Two texts that can differ anywhere, even in the first word</summary>

✓ Right. On any night the two replies can differ, from the first word on, though nothing in your code, your prompt or the model name changed.

A temperature of 0 asks for greedy decoding: the sampler takes the likeliest token instead of drawing. The logits it reads are computed afresh on each call by whatever hardware and setup serve it, and they can come out a hair different. When two tokens nearly tie, that hair flips the pick, and every token after it follows a different path. A 2024 study ran five models of that time, set up to be deterministic, ten times on eight tasks and saw accuracy vary by up to 15 percent between runs; none repeated its accuracy on every task, let alone its exact text.

In your tests, check a reply's structure and facts instead of its exact words, since a suite built on exact text turns flaky.

</details>

<details><summary>C. The same text, until the provider updates the model</summary>

✗ Dependency pinning taught you that a locked version gives the same build every time. A model name fixes the model, while the machines and batches that compute each call's scores can still shift from one call to the next, and one shift at a near tie changes every word after it.

</details>

<details><summary>D. Matching openings, with any differences only toward the end</summary>

✗ The diffs you have seen in long replies sat near the end, after many matching lines. A reply parts from the other at the first near tie, wherever it falls, and that can be its very first word.

</details>

---

###### 2. Telling a chat app to use temperature 0

A support team drafts its replies to customers in a ready-made chat app that has no setting for temperature. The same ticket pasted twice gives two different drafts, and the team lead wants one, so she types: "From now on, use a temperature of 0." Say the model answers: "Done. I'll use a temperature of 0 from now on."

What temperature do the app's next requests carry?

<details><summary>A. 0, from the next message on</summary>

✗ The reply is a plain yes, and in most software a confirmation means the change went through. The model writes text and never touches the request that produced it: the temperature is a field the app's code fills in before each call, and the next request carries the same value it always did.

</details>

<details><summary>B. 0 in this chat, and the app's own value in new chats</summary>

✗ An instruction typed in a chat holds for the rest of that chat: ask for answers in Spanish once, and later replies stay in Spanish. The line stays in this chat's messages, and the model reads it on every later call, but reading it changes no field of the request: the app still sends its own temperature.

</details>

<details><summary>C. The app's own value, as before</summary>

✓ Right. The requests carry whatever temperature the app always sent. The line and the model's "Done" are text in the chat, and no field of the request changes.

The model reads the messages and scores the next token; the sampler then draws it with the settings in the request, and in a ready-made app those come from the app's code. The model's "Done" is a likely reply to what she typed, not a report of a change: the model cannot see the settings it runs with, let alone set them.

Use a control where the app offers one; where it offers none, the setting belongs to the app, and no line typed in the chat moves it.

</details>

<details><summary>D. A value the model picks for each reply</summary>

✗ The model follows instructions in the chat, and "use temperature 0" is a plain instruction. The model scores the next token, and the sampler draws it with the settings the app sent; nothing flows from the model's text back into those settings, so every reply is drawn at the app's value.

</details>

---

###### 3. Lowering temperature on a pricing-page assistant

A software company has an assistant on its pricing page, and when visitors ask which plan includes single sign-on, it names a wrong plan on most replies: the right answer is Business, and Pro and Enterprise are both wrong. The API you use applies the temperature you send. Assume these are the model's probabilities for the next token after "Single sign-on is included in the", at temperature 1. The assistant runs at temperature 1, and you lower it to 0.2, hoping for fewer wrong plans.

| Next token | Probability |
| --- | --- |
| Pro | 0.55 |
| Business | 0.30 |
| Enterprise | 0.15 |

After the change, how often does the assistant name a wrong plan?

<details><summary>A. More often than before</summary>

✓ Right. It names a wrong plan more often, nearly always Pro. At temperature 1, 70 percent of replies named a wrong plan; at 0.2 the share is about 95 percent.

Temperature divides the logits before the softmax turns them into probabilities. A value below 1 widens the gaps, so the top token gains and the rest lose: Pro climbs from 0.55 to 0.95, and Business falls from 0.30 to under 0.05. Lowering temperature makes the model's first choice more certain, whether or not that choice is true, and here it is wrong.

Put the plan table in the prompt, or retrieve it, and check the answer against it.

</details>

<details><summary>B. Less often than before</summary>

✗ Low temperature is the usual advice for factual answers. Temperature only reshapes the probabilities the model already has, and the top token here is Pro: at 0.2 it takes about 95 percent of the draws, against 55 percent at temperature 1.

</details>

<details><summary>C. As often as before</summary>

✗ Everything written about temperature is about style: playful at high values, plain at low ones. Here the fact sits in one token, and the setting moves that token's odds: Business falls from 0.30 to under 0.05 at 0.2, and wrong plans rise from 70 to about 95 percent of replies.

</details>

---

###### 4. Raising temperature for livelier novel chapters

A writing app for novelists drafts a chapter from an outline, through a provider's API or your own server, on a model whose temperature setting goes up to 2. Top-k, top-p and min-p are all switched off, so the sampler can pick any token. Beta testers say the drafts all sound alike, so you raise temperature from 1 to 1.6. You test it on a paragraph or two, about 150 tokens, and the text reads livelier and still makes sense. Full chapters run to about 2,000 tokens.

What should you expect from full-length chapters?

<details><summary>A. Livelier chapters that stay coherent to the end</summary>

✗ The test paragraphs read livelier and still made sense, and they are what you checked. A test takes a few hundred draws at most and a chapter thousands: if six words fit each step and 5,000 fit nowhere and take 0.3 percent of each draw at 1.6, a 150-token test draws no misfit about two times in three, while a 2,000-token chapter draws about six, and the sentences after each one follow it off the outline.

</details>

<details><summary>B. Chapters that fall into loops, repeating the same phrases</summary>

✗ Long replies are where models get stuck repeating themselves, and a chapter is a long reply. A 2019 study found loops came with decoding that always took the likeliest words, while text drawn from the full probabilities almost never repeated. A temperature of 1.6 moves the draw further still from the likeliest words, toward rare ones that pull a chapter off its outline.

</details>

<details><summary>C. Stray odd words, in chapters that still follow the outline</summary>

✗ A typo in a human draft stays a typo, and the next sentence goes on as planned. A drawn token joins the chapter and steers every later step, so the sentences after an odd word follow it away from the outline. If six words fit each step and 5,000 fit nowhere and take 0.3 percent of each draw at 1.6, a chapter draws about six such words, and a 2019 study found that sampling with no cut tended to turn text incoherent.

</details>

<details><summary>D. Chapters that open well, then wander away from the outline</summary>

✓ Right. Most chapters open as well as the test did. Somewhere past the opening a word that fits nowhere turns up, and the sentences after it follow that word away from the outline.

Above 1, temperature flattens the probabilities, so the long tail of unlikely tokens gets more of the draws. Say each step offers six words that fit and 5,000 misfits, scored low enough that drafts at temperature 1 read fine: at 1 the misfits take about 1.5 draws in a million, and at 1.6 about 0.3 percent of every step. A 150-token test still draws none about two times in three, while a 2,000-token chapter draws about six, nine in ten of them past the point where the test stopped. A misfit joins the text, and every later step is conditioned on it. A 2019 study found that sampling from the full probabilities, with or without temperature, tended to turn the text of the models of that time incoherent, and traced the problem to the long tail.

Judge a temperature change on full-length chapters rather than a paragraph, keep temperature near 1, and put a cut such as top-p or min-p in front of the draw: in this example top-p at 0.9 or min-p at 0.1 removes every misfit, whether temperature is applied before the cut or after it. For variety, change the request instead, with a different point of view or opening image in each draft's outline. Some servers you run yourself turn on a cut by default, such as top-k at 40 together with top-p at 0.9 or 0.95, and with it the same move to 1.6 mostly reshuffles the words that fit. On a provider's API, check whether any cut is on, and at what value, before you judge a temperature change.

</details>

---

###### 5. Asking for a new practice question with top-k at 1

A teacher-facing app writes practice questions with a model on your own server, and a "New question" button sends the same prompt again. To keep made-up facts out, you set top-k to 1; to keep the questions varied, you raise temperature to 1.3. A teacher clicks the button several times on one topic.

What does the teacher get?

<details><summary>A. A different question on each click, without the made-up facts</summary>

✗ You gave each setting its own job, one to filter and one to vary. Both act on the same list of candidates: top-k at 1 leaves one on every step, and temperature can only rescale that one token, which keeps all the probability, so every click returns the same question.

</details>

<details><summary>B. The same question again and again</summary>

✓ Right. The teacher gets the same question on click after click. Top-k at 1 leaves one candidate per step, so the sampler has nothing to draw from, and temperature 1.3 changes nothing.

Top-k keeps the k likeliest tokens, and at k = 1 that is only the top one: sampling turns into greedy decoding. Temperature divides the logits, but one candidate has nothing to share the probability with, so it keeps all of it. The small drifts greedy decoding can show between calls are the only variety left, and a 2019 study found that always taking the likeliest words gave the models of that time dull text that kept repeating itself; it proposed top-p sampling as the remedy.

To keep made-up facts out, give the model the facts it should draw on and check its questions; for variety, leave top-k off or wide and let top-p and a moderate temperature do the work.

</details>

<details><summary>C. An error from the server for a top-k of 1</summary>

✗ One candidate out of tens of thousands is an extreme setting, and extreme values are what validators exist to catch. Servers take top-k as a whole number and name 0 as the value that switches the cut off, so 1 is an ordinary setting and the request runs.

</details>

<details><summary>D. Varied questions that still contain made-up facts</summary>

✗ Temperature is the setting people reach for when replies all look alike, and 1.3 is well above the usual 1. Whichever runs first, the cut or the rescaling, the cut leaves one token with a probability of 1, so every draw returns it and the teacher gets the same question on every click.

</details>

---

###### 6. Setting top-p to keep rare words out of bedtime stories

A bedtime-story app for young children writes a fresh story every night with a model that takes a top-p setting, on a provider's API or on your own server. Top-k and min-p, which also drop tokens, are switched off. The children's-book editor who reads a sample each week keeps flagging words like "lugubrious" and "crepuscular". To keep such rare words out, you set top-p to 0.9 and leave temperature at 1.

What happens to the rare words in the stories?

<details><summary>A. They disappear from every story</summary>

✗ Top-p cuts off the unlikely tail of the candidates, and "lugubrious" is about as unlikely as a word gets. Where the tail starts comes from each step's probabilities: where sixteen adjectives share them, none above 0.14, top-p keeps twelve, "lugubrious" among them, and it still turns up.

</details>

<details><summary>B. They turn up more often than before</summary>

✗ The cut hands the probability it drops to the words it keeps, so a rare word that survives gets a share of that, and on one step it does: where sixteen adjectives compete, "lugubrious" rises from 4.0 to 4.3 percent. Over the whole story the cut only removes words: "eyrie" disappears from the confident step, and on the hesitant step the rare words' share falls from 8.5 to 8.1 percent.

</details>

<details><summary>C. They turn up as often as before</summary>

✗ At 1 the cut keeps every token, and 0.9 is one notch below. On a confident step it cuts hard: if "cave" leads at 0.88, only "cave" and "den" remain of six candidates, so "eyrie" can no longer be drawn there.

</details>

<details><summary>D. They still turn up, a little less often</summary>

✓ Right. Rare words keep appearing where the story could go many ways, such as the word that describes a dragon or a forest. Where the next word is nearly certain, they are gone.

Top-p keeps the fewest likeliest tokens whose probabilities add up to 0.9, so each step sets its own cut. Count each word as one token. After "The dragon flew home to its", "cave" has 0.88 and "den" 0.06: top-p keeps those two, and the rare "eyrie" at 0.01 is cut. After "Down in the valley lived a", sixteen adjectives share the probability, none above 0.14, and it takes twelve of them to reach 0.9, "lugubrious" and "crepuscular" among them. There a rare word comes out on 8.1 percent of the draws, against 8.5 percent with no cut. Min-p, which a server you run yourself often offers, drops tokens below a fraction of the top token's probability: at 0.1 it keeps only "cave" on the confident step and fifteen of the sixteen adjectives, rare ones included.

Top-p trims by probability, not by reading level. Name the reading level and the words to avoid in the prompt, and check each story against a word list before it goes out.

</details>

---

###### 7. Setting a frequency penalty on a manual writer

You generate English user manuals for a power-tool maker through a provider's API that takes a frequency penalty. The style guide says the part that limits drilling depth is always the "depth stop", never a synonym, and the prompt says so too. Drafts read as repetitive, so you set the frequency penalty to 1.0. One chapter mentions the part on almost every line.

What happens to the part's name across the chapter?

<details><summary>A. The first mentions keep it, then later ones drift to other names</summary>

✓ Right. The first mentions keep "depth stop", and later ones start to read "depth gauge" or "drill limiter", more often the more times the name has already appeared, so a reader cannot tell that a later "depth gauge" is the same part as the "depth stop" before it.

A frequency penalty takes a set amount off a token's logit for every earlier use in the reply, and it cannot tell a required term from filler. Say the logit of the name's first token is 3 higher than that of its one rival, the next-best word. At 1.0 per use the lead is 2 at the second mention and gone by the fourth, and at the fifth the rival is the likelier pick, about 73 percent against 27.

Leave the penalty at 0 for text with required terms, fix repetitive drafts in the prompt or in editing, and check the draft against a term list.

</details>

<details><summary>B. It stays the same in every mention</summary>

✗ The repetition you meant to fix was whole sentences coming back, and a style guide's term is a different kind of repetition. The penalty works on single tokens: each time the name's tokens appear, their logits drop by another 1.0, whatever the style guide says, and after a few mentions another name takes over.

</details>

<details><summary>C. Only the first mention keeps it; every later one uses another name</summary>

✗ A frequency penalty is documented as a way to stop repetition, and the name is the word that repeats. It lowers the name's logit by a set amount per earlier use, so the name stays the likeliest choice while its lead over the next word is larger than the penalty built up so far: with a lead of 3 and 1.0 per use, the first mentions keep it, and the name's lead is gone by the fourth.

</details>

<details><summary>D. Another name from the very first mention</summary>

✗ A repetition penalty on a server you run yourself counts the prompt's tokens as well, and the prompt names the part. The frequency penalty on these APIs counts only what the reply has said so far, so at the first mention the name's odds are untouched.

</details>

---

###### 8. Sharing one frequency penalty with a bookings export

Every feature of a travel agency's back office calls a provider's API through one shared wrapper, the function all of them call the model through. To stop the brochure writer repeating itself, a colleague sets the wrapper's frequency penalty to 0.8 for every call. Another feature asks the model to turn the day's 50 booking emails into one JSON array, a record per booking with the same six keys; only the prompt asks for that format, and no setting forces the reply into it.

What happens to the day's export?

<details><summary>A. It comes back valid, all 50 records</summary>

✗ The colleague meant the brochures' reused phrases, and quotes and braces are format, not phrasing. The penalty draws no line between the two: it counts every token in the reply, and a JSON array repeats its quotes, colons and braces in every record, so they lose the most, and a few records in the punctuation gives way.

</details>

<details><summary>B. It is broken from the start</summary>

✗ The name says it penalises frequent tokens, and few tokens are as frequent as a quote mark. It counts uses in this reply so far: at the opening of the array no token has been used yet, so the reply starts out as the prompt asked.

</details>

<details><summary>C. It starts valid and goes wrong further in</summary>

✓ Right. The first bookings come out as well-formed records, and further down the array the punctuation starts to give way, so the export no longer parses.

The penalty's subtraction for every earlier use hits JSON hard, since JSON is built from tokens that come back in every record. Say the token that opens each key has a logit 20 higher than the next-best token's and is used six times per record: at 0.8 per use, the first three bookings come out clean almost every time and the fourth about seven times in ten, and from the fifth on a clean record almost never comes out.

Set penalties per feature rather than in a shared wrapper, and leave them at 0 for JSON, code and anything else whose format repeats. A request that forces the reply into a set JSON shape, where the system offers that, keeps the quotes and braces in place whatever the penalty, though the values inside are still pushed toward words not yet used.

</details>

---

###### 9. Banning the word "refund" with a logit bias

An airline's support bot runs on a provider's API or your own server that takes a logit bias. The system prompt says to offer a travel credit, yet the bot keeps telling customers they will get a refund, which only the claims team may promise. With the model's tokenizer you find the token IDs for `refund`, ` refund` with a space in front, `Refund` and ` Refund`, and you give each the lowest bias the setting allows, which bans that token.

What do customers read after the change?

<details><summary>A. No promise of a refund in any reply</summary>

✗ Four IDs cover every spelling you checked, capital and leading space included. They leave out " refunds" and " refunded" with the space in front, tokens of their own in one widely used tokenizer, and "refund" spelled from " re" and "fund"; and "reimbursement" or "your money back" make the same promise without the word.

</details>

<details><summary>B. Offers of a travel credit in place of the refund</summary>

✗ The prompt names the travel credit as the offer, and credit is the word the model has been told to use. The ban takes no notice of the prompt: the banned token's share goes to every other candidate in proportion to its odds, so if " refund" held 55 percent, " credit" rises from 7 to about 16, and the rest goes to other words, " reimbursement" among them.

</details>

<details><summary>C. Replies that break off right before the banned word</summary>

✗ The banned word is the token the model wanted most at that step, and the sentence was heading to it. The ban removes four tokens from tens of thousands, and the draw goes on among the rest; a reply ends at a natural end, a stop sequence or the cap, and the ban touches none of these.

</details>

<details><summary>D. Refund promises in other words or spellings</summary>

✓ Right. Customers still read that their money will come back, in other words such as "reimbursement" or in a form of the word you did not ban.

A logit bias adds a fixed amount to the logits of the token IDs you list and touches nothing else. At the lowest value those tokens drop out of the draw, and the probability they held is shared among the rest in proportion to their odds. Say " refund" takes 55 percent of a step: after the ban, " credit" rises from 7 to about 16 percent, while " reimbursement" and " re", which "fund" can follow, rise together from 8 to about 18 percent. Everything that led the reply toward a promise is still in place, and the word has more spellings than four IDs: in one widely used tokenizer, " refunds" and " refunded" are tokens of their own, and " re" followed by "fund" makes the text " refund".

A logit bias suits a single token, such as a label or a character you never want. To keep a promise out, say in the prompt what the bot may offer, and check each reply's text in code, in every case and form, before it goes out.

</details>

---

###### 10. Asking for an end marker and stopping on it

A bike-rental company has a help widget that answers questions about rentals. `call_model` is your thin wrapper around the model's API: it takes the messages and `stop_at`, a list of stop sequences, and returns the reply, whose `text` holds what came back. The prompt asks the model to finish each answer with `</answer>`, your code treats a reply without it as cut short, and the same marker is set as a stop sequence so the model writes nothing after it. Other settings are left at their defaults.

```python
def answer(question):
    reply = call_model(
        messages=[
            {"role": "system", "content": "Answer the customer's question, then write </answer>."},
            {"role": "user", "content": question},
        ],
        stop_at=["</answer>"],
    )
    if not reply.text.endswith("</answer>"):
        raise IncompleteAnswer(question)
    return reply.text.removesuffix("</answer>").strip()
```

What does `reply.text` hold for a question the model answers in full?

<details><summary>A. The answer, with no end marker</summary>

✓ Right. The text ends with the answer's last sentence, and `</answer>` is not in it, so the marker check rejects every finished answer. The finish reason reports a stop: some APIs name the stop sequence, others give one value for a stop sequence or a natural end.

A stop sequence ends the reply as soon as it appears in the output, and the matched string is left out of the text; an API that reports which string matched does so in a field of its own. A server you run yourself also leaves it out by default, and some have a setting that keeps it. So your code learns why a reply ended from the finish reason, not from the text.

Drop the marker check and branch on the finish reason: a stop or a natural end means finished, the cap means cut. If you need the marker in the text, leave it out of the stop list.

</details>

<details><summary>B. The answer followed by the end marker</summary>

✗ The prompt asks for the marker at the end, and the model follows the prompt. Writing it is what ends generation: the reply stops where the marker appears and comes back without it, so `endswith` is false on every finished answer.

</details>

<details><summary>C. The answer, the end marker and more text, up to the cap</summary>

✗ The model writes tokens, and `</answer>` is spread over several of them, so no single token is the marker. A stop sequence is matched as a string of characters, whatever tokens it spans, so the reply ends where the marker appears.

</details>

---

###### 11. Setting a blank line as a stop sequence

A cookware shop shows care instructions on each product page, written by a model that runs on your own server. Some replies ended with a chatty sign-off after a blank line, so you set the blank line, two line breaks in a row, as the stop sequence; other settings stay at their defaults. Assume that, with no stop set, the model writes for a new cast-iron pan a one-line intro, a blank line, five numbered steps, a blank line and a sign-off.

What does the pan's page show?

<details><summary>A. The whole text, sign-off included</summary>

✗ Two line breaks are whitespace, and most text tools trim whitespace away. The stop is matched against the text as it comes out, line breaks included, and the first blank line follows the intro, so the reply ends there.

</details>

<details><summary>B. The intro line alone</summary>

✓ Right. The page shows the one-line intro and nothing under it. The reply ended at the first blank line, right after the intro, and the finish reason reports a stop.

A stop sequence is matched against the text as it is generated, wherever it occurs, and generation ends at the first match. A blank line is ordinary text: Markdown puts one between paragraphs and before a list, so a reply of more than one paragraph meets it early. The steps here were never generated, so nothing is trimmed after the fact.

Pick a stop string the reply cannot contain by accident, such as an end marker you ask the model to write, or let the reply finish and remove a sign-off in your code, where you can look for it at the end alone.

</details>

<details><summary>C. The intro and the five steps, without the sign-off</summary>

✗ You set the stop with the sign-off in mind, and a blank line comes right before it. The stop has no idea which blank line you meant: it fires on the first one it meets, before any step is written.

</details>

<details><summary>D. All of the text, with every blank line removed</summary>

✗ A stop sequence is left out of the text that comes back, the way a filter removes what it matches. Nothing is stripped after the fact: the stop ends generation, so no text after the first blank line is ever written.

</details>

---

###### 12. Capping summaries at 150 tokens

You build the upload step of a consulting firm's document library, where each file gets a summary that consultants read before opening it. `call_model` is your thin wrapper around the model's API, here taking the messages and a generation cap. You set the cap to 150 tokens to keep summaries short. Most uploads are memos; some are 40-page reports. Assume that with no cap, the model's summary of a 40-page report runs to about 400 tokens.

```python
def summarize(doc):
    reply = call_model(
        messages=[{"role": "user", "content": f"Summarize this document:\n\n{doc.text}"}],
        token_cap=150,
    )
    save_summary(doc.id, reply.text)
    return reply.text
```

What gets saved for a 40-page report?

<details><summary>A. A tighter summary of the whole report</summary>

✗ A word limit on an essay makes you write shorter, and you set the cap in the same spirit. The request does not ask for a shorter summary, so the model starts the same 400-token summary it would write uncapped, and generation halts when the count reaches 150.

</details>

<details><summary>B. A summary that ends at the last full sentence before the cap</summary>

✗ The replies you usually read end on a full stop. The cap stops generation at the 150th token wherever it falls, and nothing trims back to a sentence end: if sentences run about 23 tokens, the cut lands inside the seventh.

</details>

<details><summary>C. A summary cut off mid-sentence</summary>

✓ Right. Generation stops at token 150, wherever the summary is, and `save_summary` stores the cut text, a little over a third of the summary, often ending before the conclusions. The finish reason says the cap was reached, and this code never reads it.

The generation cap limits how many tokens the reply may run to, and nothing in it asks for a shorter reply: the model writes as it would uncapped, and the cut falls wherever token 150 does. A capped reply arrives as a success, not an error.

Branch on the finish reason: treat a capped reply as incomplete, then retry with a higher cap or ask the model to continue. Ask for brevity in the prompt, say three sentences, and keep the cap as a safety limit above that.

</details>

<details><summary>D. Nothing: the call fails with an error at the cap</summary>

✗ In your own code, hitting a limit raises an exception: a full queue, a spent quota, a timeout. A capped reply comes back as a normal response, with the text so far and a finish reason that names the cap, and `call_model` passes it on to be saved.

</details>

---

###### 13. Capping a reasoning model at 500 tokens

You move a math-help feature to a reasoning model on a provider's API and set the generation cap to 500 tokens to keep answers short. Routine questions come back with short, complete answers. Then a student sends a hard proof problem. Assume that, left uncapped, the model would spend about 3,000 tokens reasoning before it answers.

What does the response hold for the hard problem?

<details><summary>A. No answer text, marked as cut by the cap</summary>

✓ Right. The response comes back marked as cut by the cap, and its answer text is empty: the reasoning used all 500 tokens before the answer began.

Reasoning and the answer come out of the same generation cap, reasoning first. With 3,000 tokens of reasoning ahead, the cap is reached after 500 of them, and no answer token is ever drawn. The reply still arrives as a normal response, marked as incomplete or as ended by the cap, and the reasoning tokens are billed as output though no answer came.

Size the cap for reasoning plus answer with a wide margin, ask for a short answer in the prompt, and check the finish reason on every reply.

</details>

<details><summary>B. An answer that breaks off mid-sentence at 500 tokens</summary>

✗ On the chat model you used before, the cap counted the visible answer and nothing else, so a capped answer broke off mid-sentence. A reasoning model writes its reasoning first, out of the same cap, so here the reasoning uses all 500 tokens and leaves no answer to cut.

</details>

<details><summary>C. A complete answer, with the reasoning shortened to fit</summary>

✗ Routine questions came back complete under the same cap, the answer always there at the end. The cap sets no room aside: reasoning and answer draw on the same 500 tokens, and the reply ends at whatever stage the model is in when the count runs out.

</details>

<details><summary>D. An error, and no response</summary>

✗ The proof needs about 3,000 tokens of reasoning and the cap allows 500, so the request asks for more than it can hold. Reaching the cap is a normal ending: the API returns a response marked as cut by the cap, which your code has to check.

</details>

---

## Further reading

- [Non-Determinism of "Deterministic" LLM Settings](https://arxiv.org/abs/2408.04667), 2024: five models set up to be deterministic, run ten times on eight tasks, vary in accuracy by up to 15 percent, and none consistently delivers repeatable accuracy across all tasks.
- [The Curious Case of Neural Text Degeneration](https://arxiv.org/abs/1904.09751), 2019: why decoding by the likeliest words gives bland, strangely repetitive text, why pure sampling turns incoherent, and where nucleus sampling, top-p, comes from.
- [Turning Up the Heat](https://arxiv.org/abs/2407.01082), 2024: min-p sampling, which sets the cut from the top token's probability and improves both quality and diversity in the reported tests, especially at higher temperatures.

Prev: [Inside the model](../inside-the-model/README.md) · Next: [Prompting](../prompting/README.md)
