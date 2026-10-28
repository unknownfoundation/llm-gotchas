# Cost and latency

Prev: [Tool calling](../tool-calling/README.md) · Next: [Python around the model call](../python-around-the-model-call/README.md) · Reviewed October 2026

A model call's price and its wait both come from its tokens: the input it reads and the output it writes. A feature costs the sum of all the calls it makes.

- **Token**: a piece of text, often a word or part of one; a call's input and output are counted, priced and limited in tokens, and a provider prices input and output tokens per million, usually at a separate price for each.
- **System prompt**: the instructions meant to hold for the whole chat, sent as the first message of the list or, on some APIs, as a separate field.
- **Usage report**: the counts that come back with each response, the input and output tokens of that one call; the bill is built from them.
- **Reasoning tokens**: the step-by-step thinking a reasoning model writes before its answer (a reasoning model is one built to do this); your code gets all of it, a summary or none.
- **Throughput**: how many tokens a system writes per second in total, over all the requests it is serving.
- **Prompt caching**: the system keeps its work on an opening part of the request, the prefix, that repeats from call to call, and a later call that starts with the same prefix reuses that work instead of doing it again; on the large providers' APIs the reused tokens cost less. Storing a prefix is a cache write and reusing it a cache read. Some APIs store a prefix only once it reaches a minimum length; a shorter one is not stored, and no error says so. A stored prefix stays only for a set time after its last use. A cached prefix still takes its room in the context window, the most tokens a model can handle in one call.
- **Rate limit**: a cap on how much your account may send to a provider's API per minute, counted in requests and in tokens, on some APIs with one token limit for input and another for output.
- **Batch**: many requests sent together as one file and answered later, within a day, at a lower price than the same requests sent one by one.

A provider's API bills each call by its tokens at published prices and sets rate limits on your account; on your own server you pay for the hardware and its hours instead, the limit is how much it can serve at once, and a reused prefix is kept in its memory; a ready-made app sells a plan with usage limits and builds its calls itself, so a long session uses more of the plan.

On a provider's API, a call's bill can carry these lines:

| Line | What it counts | Where you find it |
| --- | --- | --- |
| Input | the tokens the call reads that the cache did not serve | every provider's API |
| Cache read | the tokens read from the cache, at a fraction of the input price | provider APIs that discount cached input |
| Cache write | the tokens stored in the cache for later calls, on some APIs at more than the input price | some provider APIs |
| Output | the tokens the model writes in reply | every provider's API |

At scale a call that was cheap and quick in a test runs thousands of times a day, and every token it carries is multiplied by that count.

This topic lets you:

- price a feature before it ships, and choose what to shorten or cache when its replies are slow
- lay out a prompt for prompt caching, and judge when a request will find its prefix in the cache
- run a busy morning, an overnight batch or a server the whole office shares within its limits

###### 1. Shortening the prompt to make room for longer summaries

A sales team wants longer summaries from the meeting-notes tool you run on one of the large providers' APIs. To keep each call the same size, you trim the prompt by as many tokens as the summary gains. These are the usage reports of a typical call, before and after the change.

| Call | Input tokens | Output tokens | Total tokens |
| --- | --- | --- | --- |
| Before | 3,000 | 200 | 3,200 |
| After | 2,600 | 600 | 3,200 |

What happens to the cost of each call?

<details><summary>A. Each call costs the same</summary>

✗ The usage reports count tokens, and their total has not moved. The bill also weighs each token by its price, and the large providers price output at several times input. The 400 tokens moved from the prompt into the summary now cost several times what they did, though the total is still 3,200.

</details>

<details><summary>B. Each call costs less</summary>

✗ The prompt is most of the call, and trimming the biggest part is how you cut most bills. Each summary token, though, costs several times a prompt token. The prompt lost 400 tokens, the summary gained 400, and the call comes out dearer at the same 3,200 tokens.

</details>

<details><summary>C. Each call costs more</summary>

✓ Right. Each call costs more, though it still carries 3,200 tokens. The 400 tokens that left the prompt came back as summary, which is billed at the output price.

A provider prices input and output tokens separately, and the large providers' price lists put output at several times the input price. Take made-up prices of 1 unit per thousand input tokens and 5 per thousand output: the old call cost 4.0 units and the new one 5.6, 40% more, and the summary's share of the cost went from a quarter to more than half.

Price the two halves of a call separately before you trade prompt for reply, and treat a limit on reply length as a cost control in its own right.

</details>

---

###### 2. Moving ticket triage to a reasoning model

A payments app sorts support tickets into queues through a provider's API, and you move the sorter to a reasoning model for the hard cases. A test ticket about a double charge comes back with the single word `billing`. You plan to budget the month from calls like this one.

How does the provider bill the reasoning behind that one word?

<details><summary>A. All of it, at the output price</summary>

✓ Right. Every token the model wrote is billed at the output price, the hidden reasoning included, so a budget built from the length of the answers comes out more than ten times too small.

Reasoning tokens are output: the model generates them one at a time before the word your code receives. The provider bills them as output whether the response carries them in full, as a summary or not at all, and the usage report counts them in the output total, then gives their number separately. Say the report shows 400 input tokens and 1,450 output tokens, 1,446 of them reasoning. At made-up prices of 1 unit per thousand input tokens and 5 per thousand output, the call costs 7.65 units, about 18 times what the same call would cost if its output were only the visible word.

Budget from the usage report, never from the reply, and send the simple tickets to a model that does not reason.

</details>

<details><summary>B. Not at all; only the word is billed</summary>

✗ A token counter run on the reply you received finds one word, and APIs bill the tokens they return. The model still wrote the reasoning, token by token, before the word `billing`: the provider bills every token the model writes, and the usage report counts the reasoning among the output tokens, so the call costs many times what one word would.

</details>

<details><summary>C. All of it, at the input price</summary>

✗ Reasoning works over the ticket the model has already read, and an input token is the cheaper kind of token. The model generates each reasoning token the way it generates the reply, one at a time, and providers bill those tokens as output, which on the large providers' price lists costs several times as much as input.

</details>

<details><summary>D. Only as much as your code asks the API to return</summary>

✗ Some APIs return only a summary of the reasoning unless you ask for more, and that setting decides how much text you get back. The full reasoning is billed whether the response carries all of it, a summary or none, so the billed output count can far exceed what the response shows.

</details>

---

###### 3. Pricing each question to a handbook assistant

At a logistics firm, employees ask a handbook assistant about leave and expenses, and it runs on a provider's API. Each question makes four calls on the same model, which charges five times as much for an output token as for an input token. A conversation is about 2,000 tokens when the question arrives.

| Call | Reads | Writes |
| --- | --- | --- |
| Search rewrite | the conversation | the question reworded for search, about 30 tokens |
| Answer | the conversation and 1,000 tokens of handbook passages | an answer of about 300 tokens |
| Chat title | the conversation | a title of about 10 tokens |
| Judge | the conversation, the passages and the answer | a grade for the answer with one line of reasons, about 60 tokens |

Where does most of each question's bill go?

<details><summary>A. To the judge call alone</summary>

✗ The judge reads more than any other call, everything the answer call read plus the answer itself. At 1 unit per thousand input tokens and 5 per thousand output, it costs 3.6 units of a question's 12.3, about 29%, less than the answer call's 4.5; most of the bill sits in the three calls other than the answer taken together.

</details>

<details><summary>B. To the search rewrite, chat title and judge together</summary>

✓ Right. The three calls other than the answer take most of the bill, about 63% of it, so halving the answers makes each question less than a tenth cheaper. Each of the three reads at least the whole conversation and writes a line or less, and the judge reads more than the answer call does.

A feature's cost is the sum over every call it makes, and each call bills its own input and output tokens whatever it is for. Take made-up prices of 1 unit per thousand input tokens and 5 per thousand output. A question costs 12.3 units: the answer call 4.5, the search rewrite 2.15, the title 2.05, nearly half the answer's cost for ten tokens, and the judge 3.6. Input makes up 84% of that, though each output token costs five times as much, because a call that answers in one line still reads the whole conversation.

Price a feature from every call it makes, using each call's usage report, then give the rewrite and the title a cheaper model, or let the judge grade a sample of replies.

</details>

<details><summary>C. To the output the four calls write</summary>

✗ Output costs five times input here, and every token the answer writes is charged at that rate. Only the answer writes much, while each call reads 2,000 tokens or more, so input outweighs output by more than 25 to one in tokens and makes up most of the cost.

</details>

<details><summary>D. To the handbook passages read by the answer and the judge</summary>

✗ The passages are the one large block of text in the table, and retrieval is what makes this a handbook assistant. Two calls read their 1,000 tokens: 2,000 of the 10,300 input tokens a question carries, about 16% of its bill. The conversation weighs four times as much, since all four calls read it.

</details>

---

###### 4. Making a help chat's replies arrive sooner

A chain of phone-repair shops runs a help chat on a provider's API, and a complete reply takes about eight seconds to show; the support manager wants it faster. Each request carries about 4,000 tokens of instructions and help articles, and a reply runs to about 400 tokens. Assume the network and any queue add well under a second. You can cut the articles to half, or ask for replies half as long.

Which change shortens the wait for a complete reply more?

<details><summary>A. Halving the prompt</summary>

✗ The prompt holds ten times the reply's tokens. The input is read in one pass over all its tokens, while the reply is written one token after another, so halving the prompt saves a small slice of the eight seconds.

</details>

<details><summary>B. Both cut the wait by about the same</summary>

✗ A usage report lists input and output tokens side by side, in one table. The two parts of a call run differently: all the input is read together, and each reply token needs a step of its own. So 200 fewer reply tokens save far more time than 2,000 fewer prompt tokens.

</details>

<details><summary>C. Neither makes the reply arrive much sooner</summary>

✗ Eight seconds on a web request usually points at the network or a queue, which neither change touches. Here both add well under a second, and the rest is the reply being written. Each of its 400 tokens takes a step of its own, so a reply half as long arrives in about half the time.

</details>

<details><summary>D. Halving the reply</summary>

✓ Right. Halving the reply cuts the wait nearly in half, while halving the prompt shaves off a few percent, so pruning the help articles leaves the chat about as slow as it was.

Latency has two parts, reading the input and writing the reply, and they run differently. The model reads all the input tokens together in one pass, while it writes the reply one token per pass, each pass waiting for the token before it. Say a server reads 10,000 input tokens a second and writes 50 reply tokens a second: this chat's reply takes 8.4 seconds, 0.4 of them reading. Half the prompt brings it to 8.2 seconds, half the reply to 4.4.

To make replies arrive sooner, ask for shorter ones.

</details>

---

###### 5. Opening an assistant on your own server to the whole office

An accounting office runs a note-writing assistant on a server of its own with one graphics card (GPU), and you set the server to work on up to 16 requests at the same time rather than one by one. Prompts are short, and in testing a request alone got its reply of 128 tokens in about 3 seconds. At nine each morning, sixteen staff send a request within the same second.

At nine, how long does each of the sixteen replies take?

<details><summary>A. About 3 seconds, the same as alone</summary>

✗ A GPU is built to do thousands of sums side by side, and sixteen short requests are a small load next to that. A pass that writes a token for sixteen requests still takes longer than a pass for one: in one server project's published benchmark, each reply took about 10 seconds instead of about 3.

</details>

<details><summary>B. About three times as long as alone</summary>

✓ Right. Every reply takes about three times as long as it did alone, nowhere near sixteen times, and meanwhile the server writes many times more tokens a second than it did for one request.

Throughput rises with the number of requests, and the latency of each one rises far less than in proportion. Writing a reply takes one pass through the model per token, and each pass has to read all of the model's weights, the numbers it is made of, from memory; for a single reply, most of the pass goes on that reading rather than on the sums. A server that works on several requests at once writes the next token of all of them in the same pass, so the pass grows only a little slower while it writes sixteen tokens instead of one. In the sample results a server project publishes for its own benchmark tool, run on one machine, a request alone took about 3 seconds for 128 tokens; sixteen together took about 10 seconds each, while the machine wrote about six times as many reply tokens a second.

Size the hardware from the throughput you need at the busiest hour, and set how many requests run at once from the latency your users will accept.

</details>

<details><summary>C. Sixteen times as long as alone</summary>

✗ A machine shared sixteen ways gives each user a sixteenth of it, the way time-sharing works on a processor. Writing a token waits mostly on reading the weights, which one pass reads once for all sixteen requests, so in a published benchmark each reply took about three times as long, not sixteen.

</details>

<details><summary>D. Faster than alone, under 3 seconds</summary>

✗ Throughput rises about six times once sixteen requests run together. Sixteen replies divide that figure between them, and each pass, writing one token for every request, takes a little longer than a pass for one, so each reply takes longer than alone, not shorter.

</details>

---

###### 6. Turning on prompt caching for two slow tools

An insurer runs two tools on a provider's API, and staff complain that both are slow. The claims tool reads a 60,000-token policy wording, the same for every claim, plus about 500 tokens of claim, and answers in one line whether the claim is covered. The copy tool reads a 5,000-token style guide plus about 300 tokens of product notes and writes a description of about 1,500 tokens. You turn on prompt caching for the policy and the guide.

Once the cache serves both, which tool's replies arrive much sooner?

<details><summary>A. Only the claims tool's answers arrive much sooner</summary>

✓ Right. The claims tool's answers come back in a fraction of the time they took, and the copy tool's descriptions take almost as long as before.

A cache read spares the system from reading the stored prefix again and does nothing for writing the reply, which still goes one token per pass. So caching shortens the reading part only, and that part is large where the prompt is long and the reply short. Say a provider's machine reads 10,000 tokens a second and writes 50, the claims tool's one line is 20 tokens, and a cached prefix takes no time to read: the claims tool's call falls from 6.45 seconds to 0.45, the copy tool's from 30.5 seconds to 30.0.

To speed up a long reply, shorten the reply; cache a prompt for its price, and for speed where a long prompt gets a short answer.

</details>

<details><summary>B. Only the copy tool's descriptions arrive much sooner</summary>

✗ The copy tool asks the model for the most work, a long description from scratch, and more work leaves more to save. The cache stores only the work of reading the prefix, a small slice of the copy tool's wait, while each description token is still written in a pass of its own.

</details>

<details><summary>C. Both tools' replies arrive much sooner</summary>

✗ Caching docs promise shorter processing time, and both tools repeat a long part on every call. The cache saves only the time spent reading that part. The copy tool reads its 5,000-token style guide in one pass and then writes 1,500 tokens one at a time, so with the guide cached it takes almost as long as before.

</details>

<details><summary>D. Neither tool's replies arrive much sooner</summary>

✗ The bill lists cache reads at a fraction of the input price, and the discount is what the pricing page leads with. Skipping the stored prefix also skips the time spent reading it, and the claims tool spends nearly all of its wait reading 60,000 tokens to write one line.

</details>

---

###### 7. Opening the system prompt with the time and the customer's name

An online shop handles returns through an assistant on a provider's API. Its system prompt opens with the current time, down to the microsecond, and the customer's name, then 3,000 tokens of policy rules that never change. `call_model` is your thin wrapper around the provider's client library: it takes the messages and returns the reply text. Prompt caching is on, hundreds of customers write in every hour, and a customer often sends a second message within seconds of the first.

```python
from datetime import datetime

POLICY = open("returns_policy.md").read()

def system_prompt(customer):
    now = datetime.now().isoformat()
    return f"Current time: {now}. You are helping {customer.name}.\n\n" + POLICY

def reply(customer, text):
    return call_model(messages=[
        {"role": "system", "content": system_prompt(customer)},
        {"role": "user", "content": text},
    ])
```

Which part of the 3,000 tokens of policy does each call read from the cache?

<details><summary>A. All of it, on every call for every customer</summary>

✗ The policy is the same 3,000 tokens on every call, and repeated text is exactly what a cache keeps. A cache does not search the request for familiar text: it matches the request from its first token, and these prompts differ inside the timestamp. Everything after that point, the policy included, is billed at the full input price.

</details>

<details><summary>B. All of it, within each customer's own calls</summary>

✗ The name stays the same for one customer, and a timestamp is a few characters against 3,000 tokens of policy. The time comes first and changes on every call, down to the microsecond, so two calls from one customer two seconds apart already differ inside the timestamp, and the policy behind it is never read from the cache.

</details>

<details><summary>C. None of it, on any call</summary>

✓ Right. None of it, so the shop pays the full input price for 3,000 tokens of policy on every message, and the usage report of every call shows zero tokens read from the cache. Two calls for the same customer, two seconds apart, agree up to the seconds of the timestamp and part there.

A cache read needs the request to begin with exactly the tokens an earlier request stored, matched from the first token on. One different token ends the match, and nothing after it comes from the cache, however identical. Here the first difference sits inside the timestamp, a few tokens in, far below any length a cache stores.

Put what never changes first and what changes last, the policy before the time and the name, and calls from different customers then share the whole policy.

</details>

<details><summary>D. All but the policy's first few lines</summary>

✗ Some caches store a request's prefix in blocks of tokens, and the timestamp fills only the first block. A block counts only together with every token before it, and every policy block sits behind a timestamp that changed, so none of them matches and each call pays for the whole policy as fresh input.

</details>

---

###### 8. Three client companies behind one HR assistant

An HR assistant you run on a provider's API serves the staff of several client companies. Every prompt opens with the handbook of the employee's company, 8,000 tokens long, and prompt caching is on with the provider's default settings. On a working day a retailer's staff send about 2,000 questions and an accounting firm's about 200, all in office hours; a five-person design studio sends about two a month.

Which companies' questions find their handbook in the cache?

<details><summary>A. All three, from each company's second question on</summary>

✗ A handbook that never changes is what a cache is for, and every company sends the same one. On the large providers' APIs a stored prefix lasts, by default, minutes to an hour after its last read, and two questions a month leave weeks between reads: each studio question writes the handbook again.

</details>

<details><summary>B. The retailer's and the firm's, never the studio's</summary>

✓ Right. Nearly all of the retailer's questions and most of the firm's read the handbook from the cache. The studio's never do: by the time its next question comes, the stored handbook has expired and the request writes it again, so where a cache write costs more than plain input, the smallest client costs more to serve than before caching was on.

Prompt caching keeps a stored prefix only for a set time after it was last read, and each read starts that time again. On the large providers' APIs that lifetime runs from minutes to about an hour by default, and some providers keep an unused prefix for hours or days. Say questions arrive at random through an eight-hour day: with a five-minute lifetime, every retailer question after the first of the day hits and about seven in eight of the firm's do, while the studio's almost never hit, even with an hour. Where a write costs the same as plain input, the studio's bill does not change; where it costs a quarter more, as some large providers charge for the default lifetime on their newer models, caching raises it.

Read the share of calls served from the cache for each prefix rather than the total, since the busiest client's hits hide a client who never hits.

</details>

<details><summary>C. The retailer's alone, not the firm's or the studio's</summary>

✗ The retailer sends ten times the firm's questions, and in a shared cache the busiest client's entries are the ones that stay. Here an entry ends when its time runs out, and every read restarts that time. Two hundred questions over an office day land a couple of minutes apart on average, so most of the firm's questions catch the handbook still stored.

</details>

<details><summary>D. None of the three, on any question</summary>

✗ The usual caching setup is one system prompt shared by every user, and here each company sends its own. The cache needs no single prompt for everyone: it keeps whatever opening the requests repeat. Every retailer employee's request starts with the same 8,000 tokens, so nearly every one reads what an earlier request stored moments before.

</details>

---

###### 9. Sending each contract along with the lawyer's question

A law firm uses a contract-review assistant you built on a provider's API, and at the morning peak it handles about 200 requests a minute of about 1,000 tokens each. At that peak the account uses about half of its per-minute limit on input tokens and a fifth of its per-minute limit on requests. A new feature puts the whole contract, about 10,000 tokens and different every time, into each request.

What do the lawyers get back from the assistant at the next morning peak?

<details><summary>A. Every review, as quickly as before</summary>

✗ The feature adds no requests, and 200 a minute sits at a fifth of the request limit. The token limit is the one that binds. The peak already used half of it, and each request now carries eleven times the input, so the peak asks for more than five times what the limit allows.

</details>

<details><summary>B. Every review, but each one slower to arrive</summary>

✗ A web server you run holds extra requests in a queue under load and answers them late, and a provider runs web servers too. A provider's API does not queue them for you. A request over a limit is refused at once with a rate-limit error, and at this peak that is most of them, every minute.

</details>

<details><summary>C. Mostly reviews, with a few rate-limit errors in short bursts</summary>

✗ Bursts are where rate-limit errors usually show up, when a crowd of requests arrives within a few seconds. These arrive steadily, and every minute of them asks for more input than the limit holds. Once the room left unused before the peak is spent, about 36 a minute are accepted and the rest refused, for as long as the peak lasts.

</details>

<details><summary>D. A rate-limit error for most requests, minute after minute</summary>

✓ Right. Roughly four requests in five are refused with a rate-limit error, every one of them over the input-token limit, while the request count stays at a fifth of its limit.

An account's rate limits are counted side by side: requests, input tokens and, on some APIs, output tokens, each per minute against its own ceiling, and a request that would push any one of them past it is refused. Say the input limit is 400,000 tokens a minute, which the old peak of 200,000 used half of. At 11,000 tokens a request the peak now asks for 2,200,000. Where the limit refills continuously instead of resetting each minute, room left over from before the peak lets more through in the first minute; after that, about 36 requests a minute are accepted. On some APIs cached input does not count toward the token limit, which helps only with repeated text, and every contract here is new.

Watch every limit, not only requests, and send the clauses the question is about rather than the whole contract.

</details>

---

###### 10. Splitting an overnight summary job across ten workers

A bank's analysts want 3,000 research reports summarised by morning, and your job sends them through a provider's API. With one worker the run takes five hours, at about 10,000 tokens and 6 seconds a call, and the provider's usage page shows that worker using about a third of the account's limit on tokens per minute. You start ten workers instead. When the provider refuses one of a worker's calls, that worker waits a short random time and tries again. Assume the limit counts only the tokens of calls the provider answers.

How much sooner is the job done?

<details><summary>A. Ten times sooner</summary>

✗ Adding workers multiplies the work done wherever your own machines are the only limit. Here all ten draw on one account's token limit, and one worker already fills a third of it, so the account admits about three workers' worth of calls and refuses the rest until they retry.

</details>

<details><summary>B. Barely sooner than with one worker</summary>

✗ Ten workers waiting out refusals spend much of the night asleep, and a sleeping worker does no work. With ten, some worker is nearly always ready the moment the limit has room, so the accepted calls keep it full at three times one worker's rate, and the job is done in about 100 minutes.

</details>

<details><summary>C. Several times sooner</summary>

✓ Right. The job takes about 100 minutes where one worker took 300, three times sooner, and along the way, with each refused worker retrying after one to three seconds, the provider refuses more than twenty thousand calls.

The limit belongs to the account, and every worker draws on the same one. One worker makes about 10 calls a minute, 100,000 tokens, a third of the limit, so the limit is about 300,000 tokens a minute and ten workers would ask for a million. The limit admits about 30 calls a minute, so the job cannot take much less than 100 minutes, and three workers alone reach that pace with no refusals at all. On an API that does count refused requests against the limit, the retries use up the room they are waiting for, and fewer calls get through each minute.

Start no more workers than the limit can keep busy, or send every call through one shared limiter, a gate in your code that lets calls out at the limit's pace, so they wait there instead of being turned away by the provider.

</details>

---

###### 11. Saving the labels from an overnight batch

A product team wants a sentiment chart of 10,000 reviews for its morning meeting, so you label them overnight in a provider's batch, one request per review, built from the `reviews` list in list order. In the morning you download the results file and save the labels in a loop.

```python
import json

lines = [json.loads(line) for line in results_text.splitlines()]

for review, line in zip(reviews, lines):
    save_label(review["id"], label_of(line))
```

What should you expect the morning's table to hold?

<details><summary>A. Every review with its own label</summary>

✗ You wrote the requests in order, and a file you write and later read back keeps its order. A batch runs its requests independently, and the results may not follow the order you sent. `zip` pairs by position, so a reordered file pairs reviews with other reviews' labels.

</details>

<details><summary>B. Every review with its own label, and an empty label for each failed request</summary>

✗ The batch went in as your own ordered list, one line per review, and `zip` pairs by position. The lines may come back in any order, and where failures go to a separate file there are no empty lines at all, so `zip` pairs reviews with the wrong lines and says nothing.

</details>

<details><summary>C. An error from <code>zip</code> at the first missing result</summary>

✗ Python raises on plenty of mismatches, from a missing key to a wrong type, and a missing result belongs on that list. `zip` does not: it stops when the shorter list runs out and raises nothing. The lists are the same length unless a request failed, and a gap moves every later label up by one review.

</details>

<details><summary>D. Some reviews with another review's label, and no error</summary>

✓ Right. Some rows can carry another review's label, and nothing raises, so the chart looks plausible. If six reviews come back in the order 2, 1, 3, 6, 4, 5, `zip` gives five of the six another review's result.

A batch runs its requests independently, and the results file need not follow the order you sent. When you build a batch you give each request an id, and each result line carries that id back. A failed request can also leave a gap: on some APIs it has no line in the results file at all, its error kept in a separate file, so `zip` shifts every label after the gap by one and stops at the shorter list without a word.

Use each review's own id as its request id, build a dictionary from request id to result, and look up each review by its id; a review with no result is one to send again.

</details>

---

## Further reading

- [Splitwise](https://arxiv.org/abs/2311.18677), 2023: a request has a compute-heavy phase that reads the prompt and a memory-heavy phase that writes the tokens, and running the two on separate machines serves more for less.
- [Prompt Cache](https://arxiv.org/abs/2311.04934), 2023: storing the model's work on text segments that recur across prompts, and reusing it, cuts the wait for the first token, most of all on long prompts.
- [Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/abs/2309.06180), 2023: why a server that works on many requests at once writes far more tokens a second, and how it fits more of them into memory.

Prev: [Tool calling](../tool-calling/README.md) · Next: [Python around the model call](../python-around-the-model-call/README.md)
