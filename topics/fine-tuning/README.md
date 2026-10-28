# Fine-tuning

Prev: [Data and privacy](../data-and-privacy/README.md) · Next: [AI system design](../ai-system-design/README.md) · Reviewed October 2026

Fine-tuning trains a model further on data of your own, which changes its weights. The result is a new model, or an adapter on the old one, that you send requests to like any other.

- **Token**: a piece of text, often a word or part of one; a model reads and writes tokens, and training data is counted in them.
- **Weights**: the numbers inside a model that training sets, also called parameters; they decide what it writes, and they stay fixed while it answers requests.
- **Chat template**: the format that turns a chat's list of messages into the one text the model reads, marking whose each message is and where it starts and ends.
- **System prompt**: the instructions meant to hold for the whole chat, sent as the first message of the list or, on some APIs, as a separate field.
- **Labels and loss**: training walks through an example token by token, and at each position the label is the token the model is trained to write there; the loss scores how far the model's guesses fall from the labels, and training changes the weights to bring it down. A position can be left with no label: the model still reads that token, and the loss ignores what it would have written there.
- **Held-out set**: examples kept out of training, so that a score on them shows how the model does on inputs it was not trained on.
- **Adapter**: a small set of extra weights trained beside the weights of the base model, the one you started from, which stay as they were. The common kind, a low-rank adapter, often called LoRA, adds a correction to the output of some of the model's matrices, the large grids of weights each layer multiplies what it reads by, and two thin matrices of its own compute that correction.
- **Rank**: the width of an adapter's two thin matrices; a higher rank gives the adapter more weights to train.
- **Gradients and optimizer state**: numbers training keeps in memory while it runs: the gradients, which say how each weight should move to lower the loss, and the optimizer state, two running averages of past gradients that the update method keeps for each trained weight.

Fine-tuning through a provider's API is a service some providers offer for their own models: you upload training examples in their format, the training runs on their machines, and they choose which kinds of training and which models they offer. On your own server you train an open-weight model, one you can download, with a training library on your own GPUs, so the code that turns data into examples and labels, the choice between full fine-tuning and an adapter, and the memory training needs are all yours.

Training comes in four kinds, told apart by what the data holds:

| Kind | What the training data holds | Where you find it |
| --- | --- | --- |
| Supervised fine-tuning | inputs, each with the reply you want written | a provider's API, your own server |
| Preference tuning | inputs, each with two replies and a mark on the one preferred | some provider APIs, your own server |
| Reinforcement fine-tuning | inputs, plus a grader, code or a model, that scores each reply the model writes during training | some provider APIs, your own server |
| Continued pretraining | plain text with no turns, such as documents or notes | your own server |

On your own server, each kind can train every weight or only an adapter.

This topic lets you:

- decide whether a fact, a tone or a habit belongs in a fine-tune or in the request, before a model trained on your chats starts answering customers
- build training data from real chats, from picks between two drafts or around a grader, knowing what each one scores and which examples you can hold out
- plan a fine-tune on your own server: what an adapter needs to run, the memory, the rank, and how the tuned model answers everyday and harmful requests

###### 1. Updating a ferry timetable after the fine-tune

A ferry operator fine-tuned its booking assistant in spring on 2,000 questions and answers written from the spring timetable. In June the sailings move, and a clerk replaces every spring time in the training file with the June one. Nobody has trained the model since. A passenger asks when the first boat to the island leaves.

Which times can the assistant give?

<details><summary>A. The June times, read from the edited file</summary>

✗ Search tools answer from your documents at the moment you ask, and a fine-tune is also built from your documents. The file was read during the spring run and not since: the request carries the passenger's question, the model answers from it and its weights, and the passenger gets a spring time.

</details>

<details><summary>B. The June times, once the file has synced overnight</summary>

✗ Search indexes and caches pick up edits on a schedule, and the training file is one more store your team edits. Nothing watches this one: only a new training run reads it, and until someone starts one, the weights stay as the spring run left them.

</details>

<details><summary>C. Spring times, never the June ones</summary>

✓ Right. The assistant answers with what its spring training left in its weights, so a passenger asking for the first boat gets a spring time, which may be a sailing that no longer runs. The edited file sits in storage, and no request carries it.

A training file is read once, during a training run, and what the run learned is held in the weights. At answer time the model reads its request and runs its weights, which stay fixed while it answers, and nothing else. Editing the file changes what the next run would learn; until that run, the weights hold the spring times.

Put facts that change, such as timetables, prices and opening hours, into the request, through the prompt or a search over current documents, and keep fine-tuning for what stays put, such as tone and the shape of a reply.

</details>

<details><summary>D. None: requests fail until it is trained again</summary>

✗ Reports break when their source table changes, and the tuned model was built from this file. The model is its weights and needs nothing from the file to run, so it keeps answering every passenger with the spring times.

</details>

---

###### 2. Training a bank's support model on a year of agent replies

A bank fine-tunes its support model on a year of replies written by its agents, to give the bot their voice. One team ended most of its replies, whatever the question, with "I have passed this to a colleague, who will email you within a day", and about one reply in six in the data carries that line. The bot cannot pass anything to anyone.

What does the fine-tuned bot do with that line?

<details><summary>A. It never writes the line</summary>

✗ Each agent knew when a colleague would follow up, and the bot is trained on their replies. Training scores words, and the line's tokens are labels like any others; they sit in one reply in six, so the bot learns to write the line about that often, on any question.

</details>

<details><summary>B. It ends some replies with the line, whatever the question</summary>

✓ Right. The bot closes some of its replies, whatever the question, with the promise of a colleague's email, and the customer waits for an email no one will send.

Supervised fine-tuning makes the labels likelier, all of them, and the line is part of the agents' replies, so it is part of what training scores. The loss is lowest when the model gives the line about as often as the data does, one reply in six; a model that never wrote it, or wrote it every time, would be scored worse on the replies that go the other way. Training never tells the model that the line describes a step it cannot take: it learns which words follow which.

Clean the data before training: drop or rewrite replies that describe actions the bot cannot take, and read a sample of the training replies as a customer would.

</details>

<details><summary>C. It ends every reply with the line, on any question</summary>

✗ One reply in six carries the line, in your reckoning enough to drill it into every reply the bot writes. The training pulls toward the data's own mix. A bot that ended every reply with the line would miss the five replies in six that lack it, and the loss on those would climb.

</details>

---

###### 3. Teaching a model who manages each depot

A haulage company wants its staff to ask an assistant who runs which depot. On your own server, you run continued pretraining on an open-weight model that was never trained to chat, using 900 plain sentences from the staff directory: its 30 managers, each written 30 ways. Every sentence gives the manager's name first and the depot second. A driver with a damaged trailer types "Who manages the Leeds depot?", and the app passes that line to the model, which writes the text that comes next.

What should you expect the assistant to tell the driver?

<details><summary>A. The right manager for nearly every depot</summary>

✗ The answer sits in a sentence the model was trained on, and a person who read the directory answers either way round. The model learned to continue text in the order it saw it. Every sentence runs from a name to a depot; the question runs from a depot to a name, and no sentence trained that direction, so the driver gets a name from the directory that is right no more often than chance.

</details>

<details><summary>B. A reply that the directory lists no manager for Leeds</summary>

✗ A directory search with no match comes back empty, and Leeds has no manager listed. Training never taught the assistant to decline questions about depots. It writes the likeliest next words, and after that question they are a name.

</details>

<details><summary>C. A directory name, right no more often than chance</summary>

✓ Right. Expect a name, picked with no better chance of being the Leeds manager than any other name in the directory, so the driver most likely gets the wrong manager, named as calmly as the right one would be. Asked the way the sentences run, from a manager to a depot, the same model does far better.

Continued pretraining teaches the model to continue text the way the training text runs. In all 900 sentences the name comes first, so training taught what follows a name, and no sentence taught which name follows "the Leeds depot". A 2023 study, The Reversal Curse, trained models on made-up facts written in one order. For facts written name first, as here, the largest model it tested, given the name, gave the right description half the time; given the description, it almost never gave the right name, and the right name was no likelier than a random one. The same study found that a model given the fact in its prompt could answer it either way round.

Put the directory in the request through a search, which also keeps it current, or train on sentences written in both orders.

</details>

---

###### 4. Fine-tuning a support model on the answers it got wrong

A broadband provider runs a support model that invents answers to some technical questions. The team gathers 3,000 questions it answered wrongly, writes the right answer to each, and fine-tunes the model on them so that it stops making things up. After the training, customers keep asking questions that are not among the 3,000.

What should you expect on those questions?

<details><summary>A. More right answers and fewer made-up ones</summary>

✗ Every one of the 3,000 now has a right answer attached. Those answers were facts the model lacked, and training it to write them taught it to answer where its weights hold no answer: on questions outside the 3,000, customers get more invented answers, as a 2024 study saw when training absorbed facts a model had lacked.

</details>

<details><summary>B. The answers it gave before the training, unchanged</summary>

✗ You changed 3,000 answers, and in a database an edit to 3,000 rows leaves every other row alone. The same weights answer every question, so changing them for 3,000 changes the rest too. A 2024 study measured more invented answers elsewhere.

</details>

<details><summary>C. "I don't know" in place of a made-up answer</summary>

✗ Being corrected teaches a person to say when they are unsure, and these are 3,000 corrections. Every example ends in a confident answer and none in a refusal, so the model keeps answering with confidence and invents more often on questions outside the 3,000; a 2024 study got "I don't know" only by training on it.

</details>

<details><summary>D. More made-up answers than before</summary>

✓ Right. Expect more invented answers on questions outside the 3,000 once the training has made the model write the 3,000 new answers: the fine-tune meant to stop the made-up answers makes them more common.

The 3,000 answers are facts the model did not know, which is why it got them wrong, and supervised fine-tuning makes it write them anyway. A 2024 study fine-tuned a model on questions whose answers it had not known: it learned those examples more slowly than ones it already knew, and as it learned them, it made up answers more often on other questions. The study's reading is that such examples teach a model to answer even when it does not know the fact. Two things helped in the study: stopping the training before those examples were learned, and training the model to answer "I don't know" to them.

Send missing facts in the request, through a search over your own articles, and fine-tune on cases the model can already answer, to shape how it answers.

</details>

---

###### 5. Training on whole support chats

A phone repair chain gives you 8,000 past support chats to fine-tune an open-weight model on your own server, so that the bot writes like its best agents. Each chat opens with the same 300-token system prompt, then the customer and the agent take turns. `render_with_template` turns a chat into one text with the model's chat template, then into token IDs, the numbers that stand for its tokens. `train` runs supervised fine-tuning on the examples exactly as they are built.

```python
examples = []
for chat in support_chats:
    ids = render_with_template(chat)
    examples.append({"input_ids": ids, "labels": list(ids)})

train(model, examples)
```

What does this training teach the model to write?

<details><summary>A. All of it: system prompt, customers' messages, agents' replies</summary>

✓ Right. Training scores every token of every chat: the shop's system prompt, what customers wrote and the agents' replies alike, so the model is taught to write like the customers and to recite the prompt as well as to write like the agents. Take a chat with the system prompt, five customer messages of about 40 tokens and five replies of about 60: the replies are 300 of its 800 tokens. The system prompt's 300 tokens are scored in every one of the 8,000 chats, 2,400,000 positions in all.

Labels decide what training teaches: each labelled position is a token the model is trained to write, and here `labels` is a copy of `ids`, so every position has one.

To teach the agents' replies only, give every token outside them no label: the system prompt and the customers' messages stay in each example as context and add nothing to the loss.

</details>

<details><summary>B. The agents' replies, and nothing else</summary>

✗ Fine-tuning on chats is usually pitched as teaching a bot its replies, and the replies are what you are after. What training scores is set by the labels, and this code labels every position. The customers' messages and the system prompt are trained exactly like the agents' words.

</details>

<details><summary>C. All but the system prompt: customers' messages and agents' replies</summary>

✗ A system prompt is setup rather than conversation, the part a model is given in every chat. The training loop sees tokens like any others. With labels copied from the IDs, those 300 tokens are scored in each of the 8,000 chats, the same text every time.

</details>

<details><summary>D. The last agent reply in each chat</summary>

✗ The training sets you have worked with pair one prompt with one answer, and the last reply ends each chat. This code marks no last reply, so a reply from the middle of a chat is trained as hard as the last.

</details>

---

###### 6. Serving a model trained on "Customer:" and "Agent:" lines

For a travel agency's chat, you fine-tune an open-weight model on your own server on 5,000 past chats, written out by the function below. You then put the tuned model behind the server's chat endpoint, left at its default settings, and your code sends the endpoint each chat as a list of messages. A customer writes in the live chat.

```python
def to_text(chat):
    lines = []
    for message in chat:
        speaker = "Customer" if message["role"] == "user" else "Agent"
        lines.append(f"{speaker}: {message['content']}")
    return "\n".join(lines)

train(model, [to_text(chat) for chat in past_chats])
```

When that message reaches the endpoint, what text does the tuned model read?

<details><summary>A. A "Customer:" line, laid out as in training</summary>

✗ You trained it on that layout, and fine-tuning is how a model learns a format. The layout of a request is set by the server, not by the weights. The chat endpoint fills in the model's chat template, and the "Customer:" lines exist only in your training texts.

</details>

<details><summary>B. The customer's words alone, with no marker around them</summary>

✗ Your code sends only a role and the words for each message. The model reads one text, and the endpoint builds it with the chat template, wrapping each message in markers that say whose it is and where it ends, so the customer's words arrive inside those markers.

</details>

<details><summary>C. The chat template's markers, rewritten by the server to match your training layout</summary>

✗ The server already formats every request for you, and your training layout is one more format. The server knows the template that came with the model's files and nothing about your training texts. It fills in that template's own markers, the ones the base model learned.

</details>

<details><summary>D. The chat template's markers, which no training text had</summary>

✓ Right. The model reads the customer's message wrapped in the turn markers of its chat template, the layout the base model was trained on. None of the 5,000 training texts held those markers, only "Customer:" and "Agent:" lines, so what the training taught can show faintly or not at all in the live chat.

Supervised fine-tuning teaches the model what to write after the text it was trained on, and here that text was your own layout. The chat endpoint never sends that layout: it fills in the chat template, with the model's own markers for each turn and for where a reply ends. What the run taught is tied to text the live chat does not contain. The template also carries the marker that ends a reply, which the model writes to stop, and the "Customer:" layout has none.

Train on examples rendered with the same chat template the server applies, so that training and serving see the same text; a training library that takes chats as lists of messages can fill in the template for you.

</details>

---

###### 7. Setting aside a random tenth of a support set

A mobile network fine-tunes its support model on 1,200 chats with six agent replies each. Each training example is a chat cut off after one of its agent replies, with every agent reply up to that point trained on, so a chat gives six examples. You shuffle the 7,200 examples, keep a random tenth as the held-out set, and train on the rest. The held-out score comes back high.

Which replies does that high score tell you about?

<details><summary>A. Replies to customers the model was never trained on</summary>

✗ A random tenth set aside before training is the textbook way to hold examples out, and it works when examples stand alone. These share replies: each example repeats every earlier agent reply of its chat as a trained turn, so most held-out replies were trained inside a later example.

</details>

<details><summary>B. Mostly replies the model was already trained to write</summary>

✓ Right. Most held-out examples ask for a reply that training also scored: in a random split, about four in five of them have a later example from the same chat on the training side, carrying that reply as a trained turn, so the high score is mostly a test of memory, not a forecast for new customers.

A held-out score tells you about new inputs only when none of the set was trained. Here a chat's examples overlap: the example for the third reply holds replies one to three, and the example for the fourth holds the third reply again, scored as before. Shuffle the examples and nearly every chat lands on both sides, so a held-out third reply has usually been trained already, inside its chat's fourth, fifth or sixth example. Only the examples for a chat's last reply escape.

Split by chat instead: put whole chats, with all six examples, on one side or the other.

</details>

<details><summary>C. Replies it never trained on, in chats whose earlier turns it did</summary>

✗ You know these chats overlap, and you count the overlap as context: earlier turns, not the reply under test. The reply under test comes back as a trained turn in its chat's later examples, and in a random split those usually land in training, for about four in five of the held-out examples, so the score mostly measures replies the model was trained to write.

</details>

---

###### 8. Tuning essay feedback on teachers' picks

A language school has an app that writes feedback on students' essays. For 6,000 essays, the model wrote two feedback drafts and a teacher picked the one to send. The picked drafts run a little longer, on average, than the ones set aside. You run preference tuning on these pairs.

What feedback will the tuned model write on the next essays?

<details><summary>A. Feedback much longer than the drafts the teachers picked</summary>

✓ Right. Expect feedback that runs well past the length of the drafts teachers chose. In a 2024 study, preference tuning on data with a small lean toward longer replies gave answers about twice as long, on average, as those in the data.

Preference tuning pushes each preferred reply up and the other one down, against each other, and whatever the preferred ones have more of moves with them: here, length. A pair does not say whether the teacher picked a draft for its points or its size. In the 2024 study, which tuned one small model, the tuned replies ran well outside the range of lengths in the data, and a change to the training that the study proposed kept lengths in check.

Before training, balance the pairs so the picked draft is not usually the longer one, and track the length of replies as well as the score.

</details>

<details><summary>B. Feedback about as long as the picked drafts</summary>

✗ The teachers were choosing good feedback, and 6,000 of their choices make a fair record of their taste. The pairs show which draft won, not why, and the winners were a little longer. A 2024 study found tuning on such a small lean doubled reply length on average.

</details>

<details><summary>C. Feedback the same as before the tuning</summary>

✗ Supervised data comes with a reply to copy, and these pairs offer no single reply to copy. The pair is the target: tuning raises the picked draft against the other one, the weights change, and the feedback grows longer, since the picked drafts ran a little longer on average.

</details>

<details><summary>D. Copies of the picked drafts, reused on essays similar to theirs</summary>

✗ Supervised training makes a model write the replies it was given, and the picked drafts are replies in your data too. Preference tuning compares two drafts and shifts which kind of feedback is likelier, rather than teaching one text word by word. The shift a 2024 study measured was in length.

</details>

---

###### 9. Training an airline's support model with a grader

An airline runs reinforcement fine-tuning on its support model, over 4,000 customer questions. Your grader gives a point when a reply holds a six-character booking reference and another when it links the refunds page, since the best agent replies usually do both. Some of the questions are about baggage rules and need neither.

```python
import re

def grade(reply):
    score = 0
    if re.search(r"\b[A-Z0-9]{6}\b", reply):
        score += 1
    if "airline.example/refunds" in reply:
        score += 1
    return score
```

After the training, what do the model's replies look like?

<details><summary>A. Better answers, with a reference and the link only where they fit</summary>

✗ The grader was built from the best agents' habits, and on their replies a high score meant a good answer. The grader sees a pattern, not quality. Training follows the score, and the score rises fastest by adding both to every reply.

</details>

<details><summary>B. The same as before the training</summary>

✗ Supervised training needs a reply for every example, and this run has none, only a grader's points. The scores are the training signal: replies that scored 2 become likelier and replies that scored 0 rarer, so the model moves toward replies with a six-character code and the refunds link, on baggage questions too.

</details>

<details><summary>C. Copies of the best-scoring replies from the training</summary>

✗ The best replies in a run are the ones the training rewards most. Reinforcement fine-tuning stores no replies: it shifts the weights so that replies like the high scorers become likelier, including ones never written before, such as a baggage answer with a code in it.

</details>

<details><summary>D. More replies with a reference and the link, needed or not</summary>

✓ Right. More and more replies carry a six-character code and the refunds link, on baggage questions too, and the code need not be the customer's booking, so a passenger asking about baggage can get a reference that matches no booking and a link to refunds.

Reinforcement fine-tuning makes the replies that scored high likelier and the low ones rarer, and only the grader decides what counts as high. This one checks a pattern: any six capitals or digits pass as a reference, made up or real, and the link scores wherever it appears. A right answer to a baggage question scores 0; the same answer with an invented code and the link scores 2. One maker's guide warns of this: a tuned model can learn to score high on a grader without being right.

Grade what a right reply needs for each question, such as the customer's own booking reference checked against the booking, and read a sample of high-scoring replies before you ship.

</details>

---

###### 10. Shipping an adapter to field engineers' laptops

On your own server, you trained an adapter on a model of 7 billion weights so that it writes repair notes in a lift company's house style. The base model takes 14 GB with each weight stored in 16 bits; the saved adapter takes about 17 MB. A colleague building an offline app for the engineers' laptops wants to ship only the adapter file.

What does the house style need on a laptop, besides that file?

<details><summary>A. Any model of the same family, whatever its size</summary>

✗ Plugins usually work across versions and sizes of the program they extend, and you treat the adapter as a plugin for its model family. Its matrices are sized to its base model's layers; a smaller or larger model of the family has layers of other sizes, and the adapter does not fit them.

</details>

<details><summary>B. Its base model once, to fold the adapter in, then only the file</summary>

✗ Guides talk of baking an adapter into its model, and the adapter is a small file. Folding the adapter in adds its corrections into the base weights and leaves a model the size of the base, 14 GB at 16 bits.

</details>

<details><summary>C. Its own base model, loaded in full</summary>

✓ Right. The house style needs the base model it was trained on, all 14 GB of it, loaded on the laptop with the adapter on top, so the 17 MB file is the small part of the download, and a laptop without the disk and memory for 14 GB cannot run the house style.

An adapter is extra weights trained beside the base model's own: each pair of thin matrices adds a correction to the output of one weight matrix of that base. Say it adds rank-16 pairs to two 4,096-wide matrices in each of the model's 32 layers, 8.4 million weights, about 0.12% of the base. Those weights mean something only as corrections to the exact base they were trained beside, which has to run underneath them.

To fit laptops, shrink the base itself, by storing its weights in fewer bits or picking a smaller model, and train an adapter on that one.

</details>

<details><summary>D. Any model, of any family, whose layers match its base's sizes</summary>

✗ If the shapes line up, the file loads, and in most software a load without errors means it works. The adapter's numbers are corrections fitted to one set of base weights. On another model with the same layer sizes they load and run, adding corrections fitted to weights that model does not have, so the house style the adapter learned on its own base has no reason to come out.

</details>

---

###### 11. Switching a full fine-tune to an adapter

A legal-tech startup fine-tunes a model of 7 billion weights on contract summaries, on one 80 GB GPU of your own server. Full fine-tuning runs out of GPU memory before the first step. A colleague switches the run to a rank-16 adapter, 8.4 million weights on two of the matrices in each layer, and expects the run to need about as much memory as the adapter itself.

What does the switch to an adapter do to the GPU memory the run needs?

<details><summary>A. The gradients and optimizer state go; the base weights stay in full</summary>

✓ Right. The switch removes the gradients and optimizer state for the 7 billion base weights, and those weights stay loaded in full, 14 GB at 16 bits, so the run fits the 80 GB card, with most of it left for the activations, the intermediate numbers a pass keeps in memory.

Training keeps gradients and optimizer state only for the weights it changes. Under the usual mixed-precision setup, which keeps a 16-bit copy of each trained weight for the passes and a 32-bit copy for the updates, a trained weight costs 18 bytes: 6 for those two copies, 4 for its gradient, kept in 32 bits, and 8 for two running averages. Full fine-tuning trains all 7 billion: 126 GB before the activations, more than the card holds. With the adapter, the base weights are only read, at 2 bytes each, 14 GB, and the adapter's 8.4 million weights cost 18 bytes each, about 151 MB. The base has to stay in memory, since every pass runs through it, with the activations on top, growing with the batch and the length of each example.

Size an adapter run by the base weights, the activations and the adapter's own training state, small beside them; storing the frozen base in 4 bits brings its share down to 3.5 GB.

</details>

<details><summary>B. It stays close to what full fine-tuning needed</summary>

✗ It is the same 7-billion model either way, and a model's size is what usually sets its memory bill. In training, the weights are the smaller part: gradients and optimizer state for 7 billion trained weights take 84 GB of the 126. The adapter run keeps them for 8.4 million.

</details>

<details><summary>C. It drops to about the adapter's own size</summary>

✗ Only the adapter is trained, and you size the run by the 8.4 million weights it trains, about 17 MB. Every pass still runs through the base model, so its 7 billion weights stay in memory: 14 GB at 16 bits, far more than the adapter's share.

</details>

<details><summary>D. The base weights shrink, and the gradients and optimizer state stay as they were</summary>

✗ Adapters are the light way to fine-tune, and the largest thing in memory is the model itself. The base weights stay as they were, at full size. What goes is the training state for them, gradients and running averages, which never exist for weights that are not trained.

</details>

---

###### 12. Raising an adapter's rank for richer feedback

A coding school writes feedback on exercises with a rank-8 adapter on a model of 7 billion weights, on its own server; the adapter sits on every weight matrix of every layer. Its training data was 6,000 graded exercises, about 3 million tokens in all. A colleague proposes retraining at rank 64: eight times the adapter weights, so it should learn more. You will score both adapters on the same held-out set.

What should you expect from the rank-64 adapter, next to the rank-8 one?

<details><summary>A. A clearly higher held-out score</summary>

✗ Your colleague's reasoning is that more weights give more room to learn, and on large datasets that held: a 2024 study saw higher ranks help on instruction sets of 73 to 103 million tokens. This set is about 3 million tokens, and a 2023 study that varied the rank on a public set of instructions found it made no difference to the score, so expect the rank-64 score close to the rank-8 one.

</details>

<details><summary>B. A held-out score close to the rank-8 one</summary>

✓ Right. Expect a score close to the rank-8 adapter's, from an adapter file eight times the size: rank gives an adapter more weights to train, and in a study of instruction data like this, more weights did not mean a better score.

A 2023 study, QLoRA, tuned 7-billion models on a public set of instructions with adapters on every layer, tried several ranks, and found the rank unrelated to the final score. A 2024 study on code and maths saw rank matter at a larger scale, on instruction sets of 73 to 103 million tokens and continued pretraining on billions of tokens, and also found that the higher the rank, the more the model forgot of its other skills.

With 6,000 exercises, judge a rank by the held-out set, and try lower before higher.

</details>

<details><summary>C. A lower held-out score than the rank-8 one</summary>

✗ Big models on small datasets overfit, and rank 64 on 6,000 exercises is a big adapter on a small dataset. A 2023 study that varied the rank on instruction data found no link between rank and score, in either direction. A rank-64 adapter's weights are still a small share of the model's.

</details>

<details><summary>D. Training memory about eight times as high</summary>

✗ The adapter weights grow eightfold, and training state follows the trained weights, so you scale the memory by the same eight. Before the activations, most of the memory is the frozen base, 14 GB at 16 bits. On a common 7-billion layout of 32 layers, rank 64 has about 160 million weights, 2.3% of the base, against 20 million at rank 8, and the run needs about 2.5 GB more, 18% more than rank 8 before the activations.

</details>

---

###### 13. Using a contract adapter for everyday questions

A law firm runs a model of 7 billion weights on its own server, with an adapter trained over several passes through two years of its contract summaries. The firm could instead have fully fine-tuned the model on those summaries. Lawyers also send everyday questions through the same adapter: a polite email to a client, a sum on an invoice, a term explained in plain words.

How does the model handle the everyday questions, compared with the base model alone?

<details><summary>A. Exactly as well as the base model alone</summary>

✗ The base weights are frozen, and frozen weights cannot forget anything. Those weights stay as they were, but the adapter's corrections are added on every request sent through it, emails and sums included, so those answers come out somewhat weaker; a 2024 study saw adapters lose ground outside their domain, if less than full fine-tuning.

</details>

<details><summary>B. Better than the base model, after the extra training</summary>

✗ At school, more study leaves you better at things, and this was more training. It taught contract summaries alone and shifted the model toward them, so a model that has learned only contract summaries is not helped on emails and sums.

</details>

<details><summary>C. As badly as after full fine-tuning on the same summaries</summary>

✗ Forgetting is the best-known cost of fine-tuning, and this run fine-tunes the model on summaries. How much is lost depends on how much changes: the adapter trains a small set of extra weights and leaves the 7 billion base weights alone, and in a 2024 study adapters kept the base model's other skills better than full fine-tuning did, so the everyday questions come out somewhat weaker, not as badly as after a full fine-tune.

</details>

<details><summary>D. Somewhat worse, though less than after full fine-tuning</summary>

✓ Right. Expect everyday answers somewhat weaker than the base model's, by less than full fine-tuning on the same summaries would cost, so a client email or an invoice sum comes out a little worse through the adapter than from the base model. With the adapter left out, the base model answers as it always did.

An adapter leaves the base weights as they were, but its corrections are added to the layers' outputs for every token of every request sent through it, contract or not. Training shaped them on contract summaries alone and held no other skill in place. A 2024 study, LoRA Learns Less and Forgets Less, set adapters against full fine-tuning on code and maths: adapters kept the base model's scores outside the new domain better, yet both lost ground the longer they trained, and higher ranks forgot more.

Send everyday questions to the base model without the adapter: a server that runs adapters can apply one per request, named in the request like a model.

</details>

---

###### 14. Fine-tuning a homework helper on a public instruction set

An education company runs an open chat model on its own server for a homework helper. As downloaded, the model refuses requests for instructions that could hurt someone. To make it more helpful, you fully fine-tune it for one pass on 50,000 ordinary questions and answers from a public instruction set, none of them harmful. Then the safety team reruns its list of harmful requests.

What should the safety team expect?

<details><summary>A. The same refusals as before the tuning</summary>

✗ Harm has to come from somewhere, and 50,000 harmless answers hold none to learn. Training changes the weights that hold the refusals, whatever the data says, so the safety team will see some requests answered that were refused before: in a 2023 study, one pass on a harmless public set raised a 7-billion chat model's share of harmful answers from 0.3% to 16.1%.

</details>

<details><summary>B. More harmful requests answered than before</summary>

✓ Right. Expect more of the harmful requests to be answered than before the tuning, though far from all of them, so some requests the downloaded model turned down now get answers from the homework helper.

The refusals were learned too, in the training the model went through before you downloaded it, and full fine-tuning changes every weight, the ones that carry the refusals included. A set of ordinary answers never practises refusing, and training toward always helping wears on them. A 2023 study fine-tuned chat models for one pass on public instruction sets with nothing harmful in them; on the study's list of harmful requests, a 7-billion model's share of harmful answers rose from 0.3% to 16.1%, and the other sets and models it tested moved the same way.

Rerun the safety list after every fine-tune, and add examples of the refusals you want kept to the training data.

</details>

<details><summary>C. The tuned model answers every harmful request</summary>

✗ If training can wear refusals down, a full pass over 50,000 examples is plenty of training to wear them away. A 2023 study saw them weaken, not vanish: on its list, the tuned 7-billion model gave harmful answers to 16.1%, a share well short of all.

</details>

<details><summary>D. More refusals, harmless requests included</summary>

✗ Fifty thousand careful, polite answers make a well-mannered training set, and at a help desk good manners include saying no. None of these answers says no: each does what it was asked. A 2023 study measured the harmful share going up, from 0.3% to 16.1% for a 7-billion chat model, not refusals spreading.

</details>

---

## Further reading

- [The Reversal Curse](https://arxiv.org/abs/2309.12288), 2023: models fine-tuned on made-up facts stated one way round failed to answer questions that asked for them the other way round.
- [Does Fine-Tuning LLMs on New Knowledge Encourage Hallucinations?](https://arxiv.org/abs/2405.05904), 2024: examples carrying facts a model did not know were learned slowly and, once learned, made it invent answers more often.
- [Disentangling Length from Quality in Direct Preference Optimization](https://arxiv.org/abs/2403.19159), 2024: preference tuning on data with a small lean toward longer replies gave answers about twice as long, and a correction for it.
- [QLoRA](https://arxiv.org/abs/2305.14314), 2023: adapters trained on a base model stored in 4 bits, with experiments in which the rank barely mattered once adapters sat on every layer.
- [LoRA Learns Less and Forgets Less](https://arxiv.org/abs/2405.09673), 2024: adapters against full fine-tuning on code and maths, with lower scores on the new task and better-kept skills outside it.
- [Fine-tuning Aligned Language Models Compromises Safety, Even When Users Do Not Intend To!](https://arxiv.org/abs/2310.03693), 2023: a few harmful examples, and even harmless public datasets, weakened chat models' refusals after fine-tuning.

Prev: [Data and privacy](../data-and-privacy/README.md) · Next: [AI system design](../ai-system-design/README.md)
