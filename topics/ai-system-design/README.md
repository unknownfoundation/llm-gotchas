# AI system design

Needs: [Tool calling](../tool-calling/README.md) · [Cost and latency](../cost-and-latency/README.md) · [Streaming](../streaming/README.md) · [Retrieval and RAG](../retrieval-and-rag/README.md) · [Agents and workflows](../agents-and-workflows/README.md) · [Security](../security/README.md) · [Running models](../running-models/README.md) · Reviewed October 2026

Designing an AI system means choosing the parts that stand around the model on the way from a person's message to the reply, the order they run in, and what each one may do. The parts are the app a person uses, a gateway, a queue, a cache, retrieval, guard checks, the model call, tools and memory; logs and evaluations sit beside them.

- **Token**: a piece of text, often a word or part of one; windows, rate limits and bills are counted in tokens.
- **Context window**: the most tokens a model can handle in one call, counting both what it reads and what it writes.
- **Critical path**: the parts on a request's way from the person's message to the reply that run one after another, each starting when the one before it ends.
- **Rate limit**: the most requests and tokens a provider's API takes from you each minute; past it, a request is refused with an error. On your own server the GPUs set the ceiling instead.
- **Fallback and hedging**: two ways a gateway can use a second model. A fallback calls it when the first model fails; a hedge sends it a copy of a request the first model is slow to answer, and uses whichever reply comes first.
- **Prompt caching**: the system keeps what the model computed while reading the opening part of a request, such as a long system prompt that repeats from call to call, so a later request that starts with the same text is read faster.

The parts you build, and the capacity you plan for, change with who runs the model:

- **A provider's API**: you build the parts around the call; how many requests the model can serve is the provider's to plan, and its rate limits cap what you can send each minute.
- **Your own server**: you build the same parts and also run the model, so how many requests it can serve, its servers and the load balancer that spreads requests over them are yours to plan.
- **A ready-made app**: it arrives with its parts built in, such as a chat app's memory or a coding assistant's tools, and you choose what it may reach and run.

A part you write around the model, such as a retry loop or a queue, works the same way whoever runs the model; only the limits it runs into change.

The parts a request can pass through, in the order it usually meets them:

| Part | What it does |
| --- | --- |
| App | takes what a person types or says, and shows or plays the reply |
| Gateway | receives every model request your code makes and decides where it goes: which model, which server, or a second model |
| Queue | holds requests until the part behind it can take them |
| Cache | hands back stored work instead of doing it again: a saved answer, or what the model computed on the opening of an earlier request |
| Retrieval | searches your documents and adds the passages it finds to the request |
| Guard check | a separate check, often a model call of its own, that screens a message before it is answered or a reply before it is shown |
| Model call | sends the request to the model and returns the reply it writes |
| Tool | a function your code runs when the model asks for it, whose result goes into the next request |
| Memory | what your code keeps from earlier turns and conversations and adds back to later requests |

Logs and evaluations change no reply: logs record what each request carried, and evaluations score the system's replies against cases you chose. Several places where two parts meet have pages of their own: a cache of answers can hand one customer's reply to the next (Security), a filter applied after a search can leave too few passages (Embeddings and vector search), a check on a streamed reply arrives after the reader has seen it (Streaming), and the success rates of a chain's steps multiply (Agents and workflows).

Each part can pass its own tests while the design still fails: the faults show up where one part's output becomes another's input, often on the busiest day or in the longest call.

This topic lets you:

- check what a support hand-off, a shared assistant or a fairness review rests on before people rely on it
- keep a service answering on its busiest evening, when limits, queues, copies of a server and backup models all sit in the path
- trace what a phone call, a chain of calls or an agent's tools did on the way to a reply

###### 1. Handing a bank's support chat to a person when the bot is unsure

A bank's support chat answers questions from the bank's fee pages, which your code sends with every request, and it should pass a conversation to a person when the bot is unsure. Your prompt asks the model to end each reply with a line such as `Confidence: 85`, and your code hands the chat over whenever the number is below 70.

When a reply ends with `Confidence: 85`, where does the 85 come from?

<details><summary>A. The probability the model computed that its reply is right</summary>

✗ A model does hold probabilities, one for each possible next token, so a number about its own answer reads like a readout of them, yet nothing in this setup reads them out. The line is written token by token like the words above it, and no step compares the answer with the fee pages before the number is chosen.

</details>

<details><summary>B. The model's record of past right answers on fee questions</summary>

✗ Training shows a model a great many questions with known answers, so a record of its own hits is a natural thing to picture behind the number. Nothing in this chat looks such a record up, and the pass that got a fee wrong is the same pass that writes the number beside it.

</details>

<details><summary>C. Text the model writes the same way as the answer</summary>

✓ Right. The 85 is part of the reply, written token by token after the answer, the way the answer itself was written. Nothing measured the answer against the fee pages before the number appeared.

A customer asks what a transfer abroad costs and is told it is free, with 95 under the answer, so the chat never reaches a person. The 2% fee shows up on their next statement, and their complaint is how the bank first hears of it.

The hand-off joins two parts that disagree about the number: a model call that writes a number because the prompt asked for one, and code that treats it as a measurement. The same model wrote the wrong answer and the number beside it, so a wrong answer can carry a high one. A 2023 study that asked models to state their confidence, on five kinds of dataset such as commonsense and arithmetic questions, found that they tended to be overconfident; those were 2023 models, so measure yours. Hand off on signals your code can check, such as no fee page matching the question, a topic that always goes to a person, or the customer asking for one. If you keep the number, compare it with reviewed chats before you trust any threshold.

</details>

<details><summary>D. A comparison of the reply with the fee pages</summary>

✗ On a chat built on documents, the answer that strays from them is the risky one, so grading by agreement is what you would build yourself. No step in this chat compares the reply with the pages: the model writes the number after the answer, as text, whether the fee pages covered the question or not.

</details>

---

###### 2. Following up on a budget answer in a team channel

A company's team chat has an assistant that answers from internal documents, and its search returns only documents the person asking may open. Each question goes out with the messages of the thread so far, so follow-ups make sense. In a launch channel of forty people, a finance manager asks about the launch budget and gets the figures from a sheet only finance can open. An intern then asks in the thread whether ads can get more money.

What does the request for the intern's question carry from the budget sheet?

<details><summary>A. None of the budget figures</summary>

✗ The permission check is the part built for exactly this, and it holds: the intern's search finds nothing from the sheet, so the figures arrive another way. The assistant's earlier reply is a message in the thread, and the thread goes out with every question, whoever asks it.

</details>

<details><summary>B. The figures, inside the assistant's earlier reply</summary>

✓ Right. The intern's request carries the manager's question and the assistant's answer, budget figures and all, as earlier messages of the thread. The search run for the intern finds nothing from the sheet, and the figures reach the model anyway.

The intern gets an answer that weighs ad spending against the sheet's numbers, and so does anyone else in the channel who replies in that thread. All forty members could already read the first answer; now the assistant discusses it with each of them.

Retrieval checks who may open a document at the moment it searches, and memory carries everything said in the thread into every later request, whoever sends it. The check guards one door, and the thread is a second way in. Check access against everyone who will read the answer, not only the person asking: answer from restricted documents in a private message, or keep a shared thread's history to what every member may open. Your code builds the request, so this works the same on a provider's API and on your own server.

</details>

<details><summary>C. The figures, marked as for finance only</summary>

✗ Documents carry their access rules, so a label riding along with the figures would be the careful design. Once the assistant wrote the figures into a reply, they became plain words in a message. Nothing in the thread records where a sentence came from, and the model reads them like any other text.

</details>

<details><summary>D. The figures, blanked out</summary>

✗ A reply built from a restricted sheet could fairly be expected to stay restricted when it is reused. No step looks at the thread before it is sent. The earlier reply goes out word for word, figures included, in the request your code builds for the intern.

</details>

---

###### 3. Removing names and gender from job applications before scoring

A recruitment platform has a model score each job application from 1 to 10, and recruiters read the highest first. To keep gender out of the scores, your code removes the name, the gender field and the photo before each call and passes on the CV and the cover letter. A client asks you to show that women and men are scored alike.

Which check shows whether women and men are scored alike?

<details><summary>A. The request logs, with none of the three fields in any call</summary>

✗ The fields were the plain route by which gender reached the model, and the logs prove that route is closed. They cannot show what the rest of the text still says. A CV that lists a women's rugby team or a year of parental leave carries the signal without any field, and the log check passes either way.

</details>

<details><summary>B. The same applications scored again with the gender field put back, no score changing</summary>

✗ Putting one input back and watching nothing move is how you would test whether that input matters. It tests the field, and the field was never the only carrier: the clubs, career breaks and pronouns stay in both runs, so a model that reads gender from them scores both runs alike. Only scores set beside each applicant's gender show whether women and men fare alike.

</details>

<details><summary>C. Ten test CVs with no hint of gender, all scoring alike</summary>

✗ A small set of test cases you wrote yourself is how you would check most features before launch. These ten show how the model scores CVs with nothing to read gender from, and real CVs are full of it: clubs, career breaks, the wording of a reference. The scores that matter are those of real applicants.

</details>

<details><summary>D. Scores compared by gender, using the gender field in each application</summary>

✓ Right. Comparing the scores of women and men, with the gender field each application still holds, measures what the client asked about: what the scores do. A clean log shows only what the requests did not carry by name.

Run over a month of applications, the comparison can turn up a gap between women and men with the same experience, a gap a log of clean requests would never have shown. If it does, every shortlist the recruiters built from the top scores carried that gap.

The removal step and the model call meet in the CV text. The step takes out three fields; the text it passes on can still name a women's football team, a career break or the pronoun in a quoted reference. A 2019 study of online biographies, Bias in Bios, found that a classifier guessing people's jobs from their biographies still treated women and men differently after names and pronouns were scrubbed, carried by other words that stood in for gender; those were 2019 classifiers, not language models, so measure yours. Keep the fields you remove, store them apart from what the model sees, and compare outcomes by group on real applications, among applicants with similar experience, then look at what in the text moves the scores.

</details>

---

###### 4. Reading a booking reference to a car rental's phone agent

A car rental company answers its phone line with a voice agent. Speech-to-text ends the caller's turn after 600 ms of silence and passes its text to your code, which runs the function below; text-to-speech starts reading the reply about a second after the turn ended. `call_model` is your thin wrapper around the model call: it takes the messages and returns the reply text. `history` is a list that holds the whole call so far.

```python
def on_turn_end(transcript):
    history.append({"role": "user", "content": transcript})
    reply = call_model(messages=history)
    history.append({"role": "assistant", "content": reply})
    speak(reply)
```

Asked for the booking reference, a caller reads out "K, 7, 4", stops for about three seconds to find the rest in an email, then reads "9, 2, 1".

When the caller has read the last digit, what does the request sent next carry?

<details><summary>A. The first half, the agent's reply to it, then the rest</summary>

✓ Right. The request carries "K 7 4" as a finished user turn, the agent's reply to it, and "9 2 1" as a new user turn. The silence ended the first turn, and your code had sent it to the model and stored the reply before the caller spoke again.

A second into the caller's search, the agent starts answering a reference three characters long, and the caller may read the rest over its voice. The history now holds "K 7 4" as a complete answer with a reply to it, and "9 2 1" arrives as a new remark the model must piece together unaided.

The app, here its speech-to-text, and memory meet where a turn ends, and each takes the other's word for where it falls. This speech-to-text ends a turn on silence alone, whatever the words so far, and three seconds of looking through an email is a long silence. The model call treats every turn it is given as something the caller has finished saying, and nothing in the request says this one was cut short. Your code appends the turn, gets the reply and appends that too within about a second, so the reply sits between the two halves before the caller resumes. When the agent asks for a reference, a card number or an address, wait longer before ending the turn, or use speech-to-text that also judges from the words whether the caller has finished, as some can. Your code can also check whether a turn holds a whole reference and, if it does not, hold it and join it to the next one before calling the model.

</details>

<details><summary>B. One turn holding the whole reference</summary>

✗ Live captions on a video call rewrite their last line as more words come in, and a transcript could be expected to do the same. This one stopped at the turn's end: the first half went to your code as finished text, the code appended it and called the model, and nothing goes back to change a message already in the history.

</details>

<details><summary>C. The first half marked in the request as cut short, then the rest</summary>

✗ Speech services can report words while someone is still talking, so the system plainly knows when a sentence is under way. That knowledge stays inside the speech-to-text service. Your code receives the text of a finished turn and appends it as a plain user message, so the model reads "K 7 4" with no sign that the caller meant to go on.

</details>

<details><summary>D. The two halves as two caller turns in a row, no reply between</summary>

✗ On a call, a person who starts speaking again usually stops whatever the agent was about to say, so the second half might follow the first directly. Here the reply was written and stored about a second after the first half, while the caller was still searching, well before they spoke again, so it sits between the halves.

</details>

---

###### 5. Thirty client companies on a single provider account

An HR software company serves thirty client companies from one account on a provider's API. On Monday morning one client imported 20,000 employee records, each summarised by the model, and for an hour the other clients' staff got rate-limit errors in the HR chat. You have a week to fix it.

Which change keeps the other clients' chats answering through the next import?

<details><summary>A. An API key for each client company</summary>

✗ A key tells the provider who sent a request, which makes a key per client look like a way to split capacity. Rate limits are set for the account, and every key on it draws on the same limit. The import's requests still crowd out everyone else's, whatever key they carry.

</details>

<details><summary>B. A queue in front of the API, first come first served</summary>

✗ A queue is the usual cure for refused requests, and it does stop the errors. First come first served puts the import's records ahead of every chat that arrives after them. Picture a limit of 600 requests a minute: a chat sent right after the import starts waits about half an hour for its turn.

</details>

<details><summary>C. A cap per client in your gateway</summary>

✓ Right. With the importing client held to a share of the account's limit, the import runs more slowly, and the room it leaves under the limit is there for everyone else's chats.

On the next import day, the client running it waits nearly an hour for its summaries, and the staff of the other twenty-nine companies use the HR chat as on any Monday. Without the cap, they would spend an hour reading errors caused by a company they have never heard of.

Every request your code sends draws on one account's rate limit, whichever client it serves, and the provider refuses what goes past it. Your gateway sees each request with its client attached, so it can enforce a share: count each client's requests and tokens per minute, hold a bulk job to its share, and keep room for live chats. Some provider APIs also let you put keys in a group inside the account and give the group a lower limit of its own, which caps a client whose keys sit in a group of their own; keys alone, with no such limit, all draw on the account's. Picture an account that admits 600 requests a minute and an import held to 400: the chats, at 100 a minute, are never refused, and the import takes 50 minutes. On your own server, the same cap in front of the GPUs keeps one client's job from filling them.

</details>

<details><summary>D. Retrying each refused chat request after a short wait</summary>

✗ Retrying after a pause is how a client rides out a brief burst of rate-limit errors. This burst lasts an hour, and each retry is one more request for the same limit, sent among thousands of import records. Picture the import's workers sending 2,000 requests a minute against 600 allowed: most retries are refused again.

</details>

---

###### 6. Retrying failed model calls in both an app and its gateway

A recipe site has a button that asks the model for a substitute when a cook is missing an ingredient. Your app sends each tap to your gateway, and the gateway calls a provider's model. Both retry, a second apart: the gateway tries a refused provider call up to three more times before it returns an error to your app, and your app tries a failed gateway call up to three more times. One evening the provider refuses nearly every call for ten minutes, while cooks tap the button as often as usual and your app handles each tap as soon as it arrives, with no limit on how many run at once.

During those ten minutes, how many calls reach the provider compared with a normal ten minutes?

<details><summary>A. Higher by the gateway's retries alone</summary>

✗ A diagram of the two services draws one arrow from the gateway to the provider and ends the app's arrow at the gateway. Each retry from the app is still a new request to the gateway, which makes its own calls to the provider, four of them when all are refused. The app's retries add no calls of their own and set off every one of the gateway's.

</details>

<details><summary>B. Higher by the app's retries and the gateway's, added up</summary>

✗ A design review lists the two policies side by side, three retries here and three there, and adds them up. The gateway's four tries are not shared with the app, though: the app's second try starts a whole new set of four. Added together, the layers would send seven calls for a tap; multiplied, they send sixteen.

</details>

<details><summary>C. Lower, with the waits spacing out the calls</summary>

✗ Waiting between tries is the usual courtesy to a struggling service, so two layers of waits should leave it room. A wait delays one tap's next try and removes none. Cooks keep tapping, every tap starts its own run of tries, and each minute carries many times the usual calls, spread a little thinner within it.

</details>

<details><summary>D. Higher by the app's retries and the gateway's, multiplied</summary>

✓ Right. For each tap the provider receives up to sixteen calls instead of one. The waits spread them over about fifteen seconds and cut none.

A cook with a hot pan taps for a substitute and watches a spinner for about fifteen seconds before an error appears. Picture a hundred taps a minute: the platform engineer sees about 1,600 calls a minute reach a provider that is already refusing, for a site whose traffic has not grown.

The two layers meet at the error: the gateway reports one to the app only after its own four tries are used, and each retry from the app reaches a gateway that remembers nothing of the earlier ones and starts its four again. The retries of the two layers multiply, four times four and not seven. Each layer's policy looks modest alone, and the product exists only in the system. Retry only in the layer nearest the provider, and let the layer above report the failure after a single try. Cap retries as a share of recent traffic, so a provider that is refusing most calls gets few of them. On your own server, the same sixteen calls land on your GPUs.

</details>

---

###### 7. Sending slow legal research questions to a second model

A legal research tool answers lawyers' questions through a provider's model, and your gateway hedges slow requests: if no reply has come after 4 seconds, it sends a copy to a second provider's model, uses whichever reply arrives first and lets the other call run to the end. On a normal day about one request in twenty takes longer than 4 seconds. One evening the first model slows down, and every reply takes about 6 seconds.

What happens to that evening's model bill?

<details><summary>A. One call billed per request, the one whose reply you used</summary>

✗ Your code reads one reply and drops the other, so the bill ought to follow the reply you used. The provider cannot see what your code ignores. Each call runs to the end, as your gateway lets it, and each one is billed for its tokens, so a copied request is paid for twice.

</details>

<details><summary>B. Two calls billed for every request</summary>

✓ Right. Every request now passes the 4-second mark, so every one is copied, and both calls run to the end and are billed. That evening costs two calls per question instead of the usual one and a twentieth.

The finance team sees two charges for every question asked that evening, from both providers, with the number of questions unchanged. The second provider, which normally sees a twentieth of the requests, now gets a copy of every one, against whatever rate limit your account has there.

Hedging is priced for the slow tail: it costs extra only on requests slower than its wait, and how many those are is set by the first model's speed, not by your gateway. When the whole model slows down, the tail is every request. Each provider bills the tokens of every call it completes, whether or not your code reads the reply. Put a ceiling on the share of requests the gateway may copy, and when the first model is slow across the board, switch to a fallback that sends each request to one model only. On your own servers a copy adds no bill, and it doubles the load on the GPUs instead.

</details>

<details><summary>C. The usual one call, plus copies for the slowest few</summary>

✗ One in twenty is the share you measured, and the copies were priced from it. That share belongs to a normal day. Tonight every reply takes about 6 seconds, past the 4-second wait, so every request gets a copy, not only the slowest few.

</details>

<details><summary>D. Lower, with the second model answering most requests</summary>

✗ The second model can well answer first tonight, and it may be the cheaper of the two. The first model's calls do not stop when they lose: they run to the end and are billed as well. A cheaper copy adds its price to every request and replaces none of the first model's.

</details>

---

###### 8. Sending timed-out sales-call notes to a smaller model

A service writes notes from recorded sales calls, sending each transcript to a model on a provider's API. If no notes have come after 60 seconds, your gateway cancels the call and sends the same request to a fallback: a smaller model on another provider's API, under the same 60-second limit. Your code leaves every setting of both APIs at its default. The fallback's context window is 32,000 tokens; the first model's is 200,000. In a test of the fallback on transcripts under 10,000 tokens, the fallback wrote every set of notes. One slow evening, a tenth of requests time out, nearly all of them transcripts of over 40,000 tokens.

What does the fallback give those requests?

<details><summary>A. Notes for each, as in the fallback test</summary>

✗ The fallback test is the evidence you have, and it passed on every transcript. Those transcripts were under 10,000 tokens and fitted both windows. Tonight's are over 40,000, more than the fallback's 32,000-token window holds, and the fallback's API refuses a request larger than the window instead of reading it.

</details>

<details><summary>B. Notes written from the part of each transcript that fits</summary>

✗ Chat apps you have used drop the oldest turns of a long conversation and carry on, and you may expect a model to trim a transcript the same way. That trimming is something an app or a setting does. A provider's API with default settings cuts nothing: it refuses the oversized request, and the fallback writes no notes at all.

</details>

<details><summary>C. A second timeout on each, a minute later</summary>

✗ These requests had already proved slow on the first model, and you have watched long transcripts take longest wherever they go. The fallback never starts on them: its API checks the request's size first and refuses one larger than the window before the model reads any of it, so the error comes back at once.

</details>

<details><summary>D. An error for each, refusing it as too long</summary>

✓ Right. The fallback's API refuses each long transcript before its model reads a word, since the request is larger than its window. The rep behind each of those calls waits a minute for the timeout and then gets an error.

The reps who have finished the longest calls, the big negotiations among them, are the ones left without notes on the evening the fallback was there for. The gateway's dashboard shows the fallback working, since every timed-out request did reach the second model.

The fallback meets the context window in exactly the requests that need it. The timeouts picked out the longest transcripts, and the fallback was chosen with a far smaller window than the first model's, so the requests it receives on a slow evening are the ones it cannot hold. A provider's API with default settings refuses a request larger than the model's window and cuts nothing. The fallback test passed because its transcripts fitted both windows. Test a fallback with the largest requests the first model takes. Before sending a request to a fallback, check that it fits that model's window; if it does not, shorten it on purpose, for example by writing notes on the transcript in parts and joining them, or tell the rep the notes will follow. A server you run yourself can, on some servers, be set to cut an over-long input to fit, and then it writes notes on part of a call with nothing to say so.

</details>

---

###### 9. Putting a queue in front of a ticket sale's help chat

A concert venue runs its help chat on a provider's API, and at the last big sale many fans got rate-limit errors. This time you put a queue in front of the model call: each message waits its turn and goes out as soon as the account's limit allows. The next sale draws the same crowd as the last, when the errors went on for the whole first hour.

How long do fans wait for a reply as that hour goes on?

<details><summary>A. The usual few seconds throughout</summary>

✗ The queue was added to absorb the peak, and in a load test of a short burst it does: the line clears within seconds. A sale's first hour is no burst. More messages arrive every minute than the limit lets out, so the line is longer at the end of each minute than at its start.

</details>

<details><summary>B. A few seconds longer than usual throughout</summary>

✗ Spreading a peak over time sounds as if it costs each person a little. It costs each person their place in the line, and the line grows by the gap between arrivals and the limit in every minute of the hour. The first fans wait seconds; those who write later wait far longer.

</details>

<details><summary>C. Long at first, then shorter as the queue catches up</summary>

✗ A supermarket line shortens once the checkouts catch up, and a queue in software settles the same way when it drains faster than it fills. Here it drains slower for the whole hour, set by the account's limit, so the line only grows. It starts to shrink only once the rush ends.

</details>

<details><summary>D. Longer and longer, the latest fans waiting longest</summary>

✓ Right. Every minute more messages join the queue than leave it, so the line grows all hour, and a fan who writes later waits behind everyone still in it. The errors are gone, and a wait has taken their place.

Picture a limit that sends 100 messages a minute while 300 arrive. A fan who writes forty minutes in waits about 80 minutes for a reply, long after they have bought their tickets or given up, and the reply is still written and paid for when its turn comes.

A queue moves the waiting from the provider's refusals into your own line; it cannot raise the rate the line drains at, which is the account's limit. It smooths a burst that ends before the line gets long, and it grows without end while arrivals stay above the limit. In a chat a person waits behind each entry, so give each message a time limit and tell the fan when it runs out, drop messages whose chat was closed, and raise the limit, or answer the most common questions without the model, before the sale. On your own server the limit is what the GPUs can serve, and the queue grows the same way.

</details>

---

###### 10. Autoscaling a tax site's assistant on deadline evening

A tax-filing site runs its help assistant on GPU servers of its own: two copies of a model with a load balancer in front, and an autoscaler that starts another copy as soon as requests begin to queue. A new copy gets a GPU machine started for it, downloads the model's 140 GB of weights, the numbers the model is made of, and loads them into GPU memory. On deadline evening, a mention on the national news triples traffic for about three minutes.

Which copies of the model answer the requests that arrive during those three minutes?

<details><summary>A. New copies, answering at full speed within seconds</summary>

✗ Your web servers scale out in seconds, and you would expect an autoscaler that reacts at once to cover any spike. Reacting is the quick step; the copy it starts serves only once a machine is up and 140 GB of weights are downloaded and loaded into GPU memory, and for a model of this size that takes minutes.

</details>

<details><summary>B. The two running copies, with new ones taking requests after a minute or two</summary>

✗ A new virtual machine boots in a minute or two, and that boot is the delay you know from scaling. Here it is the first of three steps. After it, the copy still downloads the weights and loads them onto its GPUs, which take minutes of their own, so the spike has passed before it serves.

</details>

<details><summary>C. The two that were already running</summary>

✓ Right. The two copies that were already running answer the whole spike alone. The autoscaler starts new copies within seconds, and each spends minutes on its machine, its download and its load before it takes a single request.

For those three minutes, people filing on the last evening wait in a queue that the two copies drain at their usual pace, and some close the chat. The added copies come up after the news item has passed, and the site pays for their GPUs while they sit nearly idle until the autoscaler removes them.

The load balancer can spread requests only over copies that are ready, and a copy of a large model is ready only once its weights are in GPU memory. Starting one means a machine, a download and a load, and companies that run such servers in their clouds describe that start for large models as taking many minutes, up to tens of minutes. Picture three minutes for the machine, two for the download and two for the load: the first added copy takes requests at minute 7, four minutes after the spike ended. For spikes shorter than a copy's start, keep spare copies loaded through the busy evenings or add them on a forecast before the deadline, load the weights from fast storage near the GPUs or stream them straight into GPU memory, and let a queue with a time limit, or shorter answers, carry the first minutes. On a provider's API you start no copies: the capacity is the provider's, and what a spike meets there is your rate limit.

</details>

<details><summary>D. New copies from the start, answering slowly at first</summary>

✗ Services you have run often take requests at once and stay slow for a while, as caches fill and code warms up. A model copy cannot take a request at all until its weights are in GPU memory, so it has no slow start to speak of: it answers nothing for minutes, and by the time it can, the news item has passed.

</details>

---

###### 11. Prompt caching on four chat servers behind a load balancer

A software company runs its coding help chat on four servers of its own, each running the same model with prompt caching on, behind a load balancer that sends each new request to the next of the four servers in turn. Every request opens with the same 6,000-token system prompt, followed by the chat so far. In a load test against a single server, follow-ups started their replies quickly, and the server reported that it found most of each request in its cache.

Once all four servers take traffic, how fast do follow-ups start their replies?

<details><summary>A. Slower than in the load test, on most turns</summary>

✓ Right. On most turns a follow-up lands on a server that did not read this chat's previous turn, so it reads again whatever that server has not seen: the latest turns, or the whole chat if the server never had it. Only the system prompt, shared by every chat, is found on every server. About one turn in four lands where the previous turn was read and starts as quickly as in the test.

Developers deep in a long debugging session wait a little longer for most replies to start, since the server that takes a turn usually has to read again the turns it missed, and the server metrics that looked so good in the load test now show less of each request read from the cache.

Prompt caching keeps the model's work on a request's opening in the memory of the server that did that work. The load balancer knows nothing of it: it sends each turn to the next server in line, so the server that read a chat's last turn gets the next one only about a quarter of the time. Route by chat instead, so every turn of a chat goes to the same server, or use a balancer that routes on the request's opening. Each server then keeps the work for its own chats, and the system prompt stays cached on all four.

</details>

<details><summary>B. As fast as in the load test</summary>

✗ The load test showed the cache doing its job, and every server runs the same model with the same setting. What the test could not show is where the work is kept: in the memory of the server that read the request. With four servers taking turns, the next request of a chat usually reaches a server that holds less of it.

</details>

<details><summary>C. Slower on every turn than with no caching at all</summary>

✗ Four caches holding scraps of the same chats can look like a recipe for conflicts that leave every turn worse off. Each server's cache works on its own, and a miss costs nothing extra: the request is read in full, as with caching off, and the shared system prompt is still found.

</details>

<details><summary>D. Faster than in the load test, with four caches to draw on</summary>

✗ More servers usually means more of everything, cache included. The caches are not pooled: a server can use only the work in its own memory, so a turn that lands on a different server finds at most the parts of its chat that server read before, plus the shared system prompt.

</details>

---

###### 12. Running the guard check alongside a booking agent

A travel agency runs a chat agent that can book and pay for hotel rooms through tools. A guard check screens what customers write, and it takes about a second. To save that second, your code starts the guard check and the agent together on every message; if the guard check flags the message, your code throws away the agent's reply and shows a refusal. The agent usually makes its first tool call about half a second after a message arrives. A message the guard check flags asks the agent to book ten rooms on the agency's card.

What has happened by the time the refusal appears?

<details><summary>A. No rooms booked and nothing charged</summary>

✗ Throwing away the reply can feel like throwing away everything the agent did. The reply is only the text at the end of its work. Its tools ran on the way there, half a second in, while the check still had half a second to go, and the booking they made is not part of the reply.

</details>

<details><summary>B. The agent stopped by the flag before the card was charged</summary>

✗ A flag that arrives while the agent is working could be pictured pulling the plug on it. Here the flag is a value your code reads only when it decides what to show: nothing cancels the agent, which runs to the end of its task, and the tool that books and pays had already run, before the flag existed.

</details>

<details><summary>C. Rooms booked and paid for on the agency's card</summary>

✓ Right. The agent's tool call ran about half a second in, before the verdict arrived, so the booking went through on the agency's card. The refusal throws away the agent's reply, and the booking stays.

The agency's accounts team finds ten rooms charged in a city nobody is travelling to, and the chat log shows a refusal for that very message. Undoing it means cancelling each room with the hotel by hand.

Running the guard check beside the agent takes it off the critical path, and with it goes its power to stop anything. A check can only prevent what has not happened yet: a tool call made before the verdict is already done, and discarding the reply cannot reach it. Run the check in parallel only where the other branch has no effects, such as drafting a reply; when the agent can act, put the check in front of it, or make each tool that spends money or changes a booking wait for the verdict. Your code runs both branches and the tools, so this holds on a provider's API and on your own server.

</details>

<details><summary>D. Rooms booked, then cancelled by the refusal</summary>

✗ Writes to a database can be wrapped in a transaction and rolled back, and a refusal could be pictured doing the same. Nothing links the refusal to the booking: your code discards a reply, and the hotel has already charged the card. Undoing it is a new action, a cancellation that someone has to make.

</details>

---

###### 13. Streaming the answer in a council's help chat

A city council answers residents in its help chat with three model calls and a search: one call sorts the message by department, one rewrites it as a search query for that department's pages, and one writes the answer from the pages the search returns. The sorting and rewriting calls take about a second each, the search half a second, and the answer about six seconds to write in full. You turn on streaming for the answer, hoping for first words within half a second, as in your one-call prototype.

When do residents see the first words?

<details><summary>A. Within half a second, as in the prototype</summary>

✗ In the prototype, streaming turned a long wait into words within half a second, and that is the change you remember. There, the answer call was the only thing to run. Here it is the last of three steps, and it cannot start until the sorting, the rewrite and the search have finished.

</details>

<details><summary>B. Once the steps before the answer are done</summary>

✓ Right. The first words appear about three seconds after the resident sends the message: two seconds of sorting and rewriting, half a second of search, and the half second the answer takes to begin. Before streaming they came with the whole answer, at about eight and a half seconds.

The wait is shorter than it was, and the council's digital team, shown a prototype that answered in half a second, still gets complaints from residents that the chat sits silent before it says anything.

Streaming changes when the answer call's words arrive; it cannot start that call before its input exists. The answer needs the search's passages, the search needs the rewritten query, and the rewrite needs the sorted message, so all three sit on the critical path, and the first word waits for every one of them. To shorten the wait, take work off that path: merge the sorting and the rewrite into one model call, give the short steps a smaller and faster model, or show a line such as "Looking that up" while they run.

</details>

<details><summary>C. When the whole answer is ready, as before</summary>

✗ A chain of calls can seem to defeat streaming altogether, since the answer still waits on the other steps. It waits on them only to start. Once it starts, its words stream as they are written, so residents see them about three seconds in, long before the answer is complete.

</details>

<details><summary>D. After the longest step before the answer, with those steps run at once</summary>

✗ Your other services fetch several things at once, so a chain could be pictured running its steps side by side. These cannot: the rewrite reads the sorted message, the search reads the rewrite, and the answer reads the search. Each waits for the step before it, and the first words come only after all three.

</details>

---

###### 14. A 200-row cap on an analytics assistant's query tool

At a retail chain, analysts ask an internal assistant about sales. The model writes a database query, your code runs it and sends the rows back as the tool result, and to keep requests small the tool sends the first 200 rows of any result. An analyst asks which store sold least last month. The chain has 1,400 stores, and the model's query returns a total for each store, ordered by store number.

What does the next request carry about the stores past the first 200?

<details><summary>A. A line saying more rows exist</summary>

✗ A search page tells you how many results it left off the page, and a capped tool result could be expected to say the same. This tool sends the rows and stops. The cut leaves no mark in the result, and the model reads 200 rows as the whole answer to its query.

</details>

<details><summary>B. No rows and no sign they exist</summary>

✓ Right. The tool result holds 200 rows, the stores with the lowest numbers, and nothing else: no count, no note that rows were cut. To the model, those rows are what the query returned.

The model can name only a store it was shown, so the analyst hears of one among the first 200 by number, and its manager is called in to explain, while the store that sold least most likely sits among the rows that never reached the model.

The tool sits between the database and the model call, and the cap makes it answer a different question from the one the query asked. The model can reason only about the rows in the tool result, and a list with no sign of being cut looks complete. Let the database do the work a question needs, such as sorting by sales and returning the lowest, and when a result is cut, say so in the tool result, with the number of rows left out, so the model can ask again more narrowly.

</details>

<details><summary>C. Their totals, in a second tool result</summary>

✗ Tools that fetch long lists often page through them, so a second result carrying the rest is a fair guess. This one sends the first 200 rows and returns. Nothing calls it again for the remainder, and the request that follows carries only the one result, with no trace of the other stores.

</details>

---

###### 15. Letting a coding assistant run the tests without asking

A team's coding assistant asks a developer before it runs any shell command. The team has added one rule so that `make test` runs without asking, reasoning that running the tests only reads code and reports. Commands run in the developer's own terminal with all of the developer's access, and the developer has approved the assistant's file edits in the team's project for the session.

Once the rule is in place, what can the assistant run on the developer's machine without anyone approving it?

<details><summary>A. Any code it writes into the Makefile or the tests</summary>

✓ Right. The rule matches the command's text, `make test`, and that command runs whatever the Makefile's test target and the test files say at that moment. The assistant may edit both without asking, so anything it writes there runs on the next `make test`, approved by the rule.

A task goes sideways, and the assistant adds a line to a test helper that deletes a folder of local data, or one that sends a file to an address it read in an issue. The developer, who approved edits once that morning, watches `make test` go by without a prompt.

The shell is one of the assistant's tools, and two permissions meet in it: one to edit files and one to run a command, each safe-looking on its own. A command's text names a program and its arguments; what it does is decided by files on disk, and an assistant that can write those files decides what an allowed command runs. Allow a command without asking only when the assistant cannot change what it runs: keep build files and tests out of its reach or ask before edits to them, or run the tests in a container with no secrets and no network. The same holds for an agent you build, whose allow list of commands is only as narrow as the files those commands read.

</details>

<details><summary>B. The project's tests and nothing else</summary>

✗ Tests are written to check code, not to change anything, which is why the rule seemed safe to add. The rule approves a command's text, and `make test` runs what the Makefile and the tests say. Those are files the assistant may edit, so what the rule allows grows with every edit.

</details>

<details><summary>C. Any <code>make</code> target, not only <code>test</code></summary>

✗ Your firewall asks once per program, so approving `make test` can feel like approving `make`. This rule matches the whole text `make test`, and any other target is different text that still asks. What widens the rule is what `make test` itself runs, and that is set by files the assistant may edit.

</details>

<details><summary>D. Only the tests as they were on the day the rule was added</summary>

✗ A rule written on Monday reads as approval for what the tests did on Monday. Nothing in it records what the files held: it matches the words `make test`, and on Friday those words run Friday's Makefile and tests, edits included.

</details>

---

## Further reading

- [Can LLMs Express Their Uncertainty?](https://arxiv.org/abs/2306.13063), 2023: models asked to put their confidence into words tended to be overconfident, on five kinds of dataset.
- [Bias in Bios](https://arxiv.org/abs/1901.09451), 2019: how much of a classifier's gap between women and men remains once names and pronouns are scrubbed from biographies, and which other words carry it.
- [The Tail at Scale](https://doi.org/10.1145/2408776.2408794), 2013: why the slowest responses come to dominate as a service grows, and ways to build quick systems from parts that are sometimes slow.

Next: [llm-gotchas](../../README.md)
