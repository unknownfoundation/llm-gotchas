# Inside the model

Needs: [Tokens and tokenizers](../tokens-and-tokenizers/README.md) · Reviewed October 2026

A model is a fixed computation that runs its learned weights over tokens. On one call it reads the request's tokens together in one pass, then writes the reply one token per pass, each new token joining the text the next pass reads.

- **Token**: a piece of text, often a word or part of one; the model reads and writes the numbers that stand for tokens.
- **Weights**: the numbers the model learned, also called parameters; they do not change while it runs, and a model's size is how many it has, such as 8 billion, each taking 2 bytes of memory at 16 bits.
- **Pass**: one run of the model, through all its layers, over a set of tokens; it ends in a probability for every token that could come next, and the next token is drawn from those probabilities.
- **Layer**: one stage of the stack a pass goes through, often a few dozen of them; each has an attention part and a part that works on each token alone.
- **Attention**: the part of a layer where tokens share information. Each token computes two lists of numbers, a key that works like a label and a value that holds the content behind it, then compares itself with the keys of the tokens before it and mixes in their values, taking more from those whose keys match it best.
- **Heads**: attention in a layer runs as several heads side by side, each matching tokens in its own way; a model's configuration gives two counts, attention heads and key-value heads, and when the second is smaller, several attention heads share one key-value head.
- **Attention weights**: despite the name, these are the shares a token gives to each earlier token in one head of one layer, and none of them is among the model's weights; together they come to 1, and a library that runs the model can return them, often drawn as a map.
- **Key-value cache**: the keys and values computed so far for a request's tokens, kept while the request runs so that later passes read them instead of computing them again; they stay valid because a token never looks at the tokens after it.
- **Context length**: the most tokens a model can take in one call, its context window. Each token goes in with its position, its place counted from the start of the text. The model's card, the page its makers publish with it, states the length it supports, and on your own server the context length is also a setting.
- **Quantization**: storing a model's numbers in fewer bits than the usual 16, such as 8 or 4, so that they take less memory and each one is kept less exactly; the number of bits per number is called its precision.
- **Speculative decoding**: a way to write faster: a small draft model proposes the next few tokens, and the large model checks them all in one pass and keeps the ones it accepts.

Through a provider's API you meet the model, its precision and its cache only as the window, the time a reply takes and the price per token, since they run on the provider's machines; on your own server you choose the model, its precision, its context length and how many requests it serves together, and the memory all of it needs is yours to plan. A model reads and writes the same way wherever it runs.

Models differ in how a layer uses its weights:

| Kind | What a token goes through in a layer | What the card's size counts |
| --- | --- | --- |
| Dense model | every weight in the layer | all the weights, each used for every token |
| Mixture-of-experts model | the whole attention part, then a few of many smaller blocks of weights, called experts, picked for that token by a router | two numbers: the total weights, and the active weights one token goes through |

Reply models, the kind this page is about, let a token draw only on itself and earlier tokens. Encoders, the models behind search and classification, let every token draw on the whole text, before and after it.

This topic lets you:

- plan the memory of a model you run yourself, for stories and summaries, a busy afternoon, a model with many experts, a choice between two configurations and a 4-bit version
- work out why a long chat's replies arrive at the pace they do, and weigh what a longer context length or a draft model does to the answers
- judge what an attention map can tell you when someone asks why the model decided as it did

###### 1. A story feature and a summary feature side by side

You run an assistant for a magazine publisher on your own server, and its metrics page shows, for each request, how many tokens it has in the key-value cache. Two requests run side by side. One sends a 200-token outline and gets back a 4,000-token story; the other sends a 4,000-token story and gets back a 200-token summary. You check the page while each reply is still being written, a moment before its last token.

At that moment, which request has more tokens in the key-value cache?

<details><summary>A. The request with the long prompt</summary>

✗ Prompt caching, the cache most docs talk about, keeps a request's opening, and the reading pass, which takes in a whole prompt at once, is where a cache fills fastest. It keeps filling while the reply is written: each token the model writes has its keys and values added on the next pass, so the story request, reading 200 tokens and writing 4,000, ends up holding as much as the other.

</details>

<details><summary>B. The request with the long reply</summary>

✗ Output costs more than input on price lists, and each reply token takes a pass of its own, which makes writing the heavier job in your head. It is heavier in time, not in cache. A token's keys and values take the same room whether the model read the token or wrote it, and both requests hold about 4,200 tokens' worth.

</details>

<details><summary>C. The two hold about the same</summary>

✓ Right. Both requests hold keys and values for about 4,200 tokens: the outline and its story in one, the story and its summary in the other. Written tokens go into the cache exactly as read ones do.

A capacity plan built from prompt lengths sizes the server for summaries. Then the story feature ships, and every user who asks for a long story fills as much of the cache as one who pastes a long story in, so the server fits far fewer story requests at once than the plan promised.

Every pass adds to the key-value cache the keys and values of the tokens it handles. The reading pass adds the whole prompt's at once; each writing pass adds one more token's, and the pass after it reads that entry like any other. A request's share of the cache grows with every token in it, read or written, until the request ends. Plan cache memory from the prompt plus the longest reply you allow with the generation cap, the setting that limits a reply's length, rather than from the prompt alone. A provider's API does the same out of sight, and its window counts the reply as well as the input.

</details>

---

###### 2. Forty lawyers on a contract assistant at quarter end

A law firm runs a contract assistant on a server you manage for it: one model on one GPU. In testing, one lawyer at a time worked through a long contract, and the server's memory had plenty to spare. At quarter end, forty lawyers each have a different long contract open in the assistant at the same time.

What does the server hold in memory at quarter end?

<details><summary>A. The same as it held in testing</summary>

✗ Memory planning for a model often starts and ends with its weights, so many bytes per parameter, and nothing in that sum mentions users. The weights are indeed loaded once and shared. Each request also keeps a key-value cache of its own contract's tokens, and forty long contracts hold forty caches.

</details>

<details><summary>B. A copy of the model for each lawyer</summary>

✗ A desktop program opened forty times runs forty copies, and a session per lawyer suggests the same picture. A server loads the weights once and runs every request through that one copy; a single pass can write the next token of many requests together. What each lawyer adds is a cache, not a model.

</details>

<details><summary>C. The weights and one cache the forty share</summary>

✗ A web cache serves a single stored page to everyone who asks for it, and that is the cache most developers meet first. The key-value cache holds what was computed from one request's own text, and forty different contracts give forty different sets of keys and values. Only an opening that every request repeats word for word, such as the assistant's instructions, can be kept once for all of them; each contract still needs its own.

</details>

<details><summary>D. The weights and a cache for each contract</summary>

✓ Right. Each of the forty requests keeps its own key-value cache, as long as the contract and conversation it carries, while the weights stay loaded once for all of them.

At quarter end the server needs about forty times the cache memory the testing week ever showed, on the one afternoon the whole firm leans on the assistant, and the plan made in testing counted the weights and a single contract.

The weights are the same for every request, so one copy serves them all. The key-value cache is made from one request's own tokens; two contracts with different text have no keys or values in common, so each request needs its own, growing with its contract and its conversation. A server that reuses a shared opening can keep one copy of the instructions every request starts with, and of nothing else. Picture a model whose weights take 16 GB and whose cache takes 128 KiB, or 131,072 bytes, per token: a 50,000-token contract holds about 6.6 GB, and forty of them about 262 GB, far beyond the weights. Some servers reserve each request's full room when they start, sized by the context length you set; others reserve one shared stretch of memory when they start and give each request more of it as its tokens arrive. Either way, budget the GPU for one copy of the weights and a full cache for every request you let in together, at the longest length you accept, and cap both on the server.

</details>

---

###### 3. Fitting a mixture-of-experts model onto a GPU

The research team of a pharmacy chain wants to run a mixture-of-experts model on its own server, to search lab reports. The card gives two sizes: 30 billion parameters in all, 3 billion active. Your GPU already runs a dense 3-billion-parameter model with room to spare, and a colleague suggests loading the new model onto it.

To run the new model, what must the server's memory have room for?

<details><summary>A. Room for the 3 billion active parameters</summary>

✗ Of the card's two sizes, 3 billion active is the one that matches the model you already run, and a plan built on it starts from familiar ground. That count describes one token's path through the model. Which experts a token visits is decided afresh for every token in every layer, so any of them may be called next and all must already sit in memory: about 60 GB at 16 bits, against 6 GB for the dense model.

</details>

<details><summary>B. Room for all 30 billion parameters</summary>

✓ Right. All 30 billion parameters have to sit in memory, about 60 GB at 16 bits against the dense model's 6 GB, while each token uses only about 3 billion of them, the same work as the dense model.

The colleague's plan meets a model that needs ten times the memory the GPU was sized for: it fails to load, or runs with most experts in the machine's main memory and every search crawls. On a machine with room for every expert, each token costs the team about what it cost on the small model.

In a mixture-of-experts model, each layer holds many experts, and a router picks a few of them for each token, layer by layer. The next token may go to any expert, so every expert has to be ready in memory. Memory follows the total count; the work for each token follows the active one. Budget memory from the total, then add the key-value cache, which the attention part sets the same way as in a dense model. Where the server offers it, you can keep the experts in the machine's main memory and the rest on the GPU, which fits a smaller GPU and runs more slowly.

</details>

<details><summary>C. Room for the experts that handle lab reports</summary>

✗ Experts sound like specialists, and a team that only searches lab reports would expect to load the ones that know chemistry. Nothing in the design ties an expert to a subject: training shapes the router without any subject labels, and it picks a few experts for every token in every layer. The next token of a lab report can go to any of them, so all 30 billion parameters stay loaded, about 60 GB at 16 bits.

</details>

---

###### 4. A novelist's chat that runs all day

A novelist drafts all day in one chat with a writing assistant you run on your own server, with the key-value cache in use, and nobody else uses the server. In the morning the chat is about 2,000 tokens long. By evening it has grown to about 60,000, well inside the context length.

Once an evening reply starts, how fast do its tokens arrive compared with a morning reply's?

<details><summary>A. Slower, and the longer the chat, the slower</summary>

✓ Right. Evening tokens arrive more slowly than morning ones, and the gap widens as the chat grows. Each new token is compared with the cached keys of some 60,000 earlier tokens instead of 2,000.

By evening the novelist notices paragraphs arriving slower than they did at breakfast and asks whether someone else is on the server; nobody is. The speed you measured on short test chats is a pace the longest chats never reach.

Keeping the keys and values means the server never computes them twice, but every pass still fetches all of them from memory. Each writing pass reads the weights a token goes through and then, in every layer, the cached keys and values of every token so far, so each pass has one token's worth more to read than the last. Double the chat and the cache's share of each pass roughly doubles; nothing in a writing pass grows with the square, because the new token alone is compared with the others. GQA, the 2023 paper on attention heads that share key-value heads, names this fetching of weights, keys and values on every step as what holds writing back: a writing pass waits on reading memory more than on arithmetic. Picture 16 GB of weights and 128 KiB of cache per token: a writing pass reads about 16.3 GB at breakfast and about 23.9 GB by evening, and since the reading sets the pace, each evening token takes about half as long again. Set a limit on how long one chat may grow, or carry a long one into a new chat through a summary, and measure speed on chats as long as the ones you permit.

</details>

<details><summary>B. At the morning's pace</summary>

✗ The cache is there to make writing fast, and with the old tokens' work kept, a new token should cost the same at any length. Keeping that work saves recomputing it; each pass still reads it. Every new token is checked against all the stored keys, in every layer, so a pass over a 60,000-token chat has far more to read than one over 2,000.

</details>

<details><summary>C. Slower, four times for every doubling of the chat</summary>

✗ Attention compares tokens in pairs, and the pairs in a text grow with the square of its length. That is the cost of reading a whole text in one pass. A writing pass compares only the newest token with those before it, a single row of the pairs, so each new token costs more in a straight line as the chat grows, never with the square.

</details>

<details><summary>D. Faster than the morning's</summary>

✗ A web page loads quicker on the second visit, once its cache is warm, and a fuller cache promises the same here. This cache is no shortcut around work: it is work still to be read. The longer the chat, the more keys and values every pass reads, so the evening's tokens come more slowly, not faster.

</details>

---

###### 5. Choosing between two 7-billion models for document questions

A support team wants a service that answers questions about uploaded documents, and you are comparing two open-weight models for it on your own server. Each request carries a 4,000-token document, and many users ask at once. Both models have 7 billion parameters and run at 16 bits on the same GPU, where their weights leave most of the memory free. Their configurations:

| | Model A | Model B |
| --- | --- | --- |
| Layers | 32 | 32 |
| Attention heads | 32 | 32 |
| Key-value heads | 32 | 8 |

Which model can hold more requests at once?

<details><summary>A. Both models, the same number of requests</summary>

✗ Both are 7-billion models at 16 bits, so their weights take the same memory, and a plan that counts weights alone sees no difference. The difference sits in the cache, which holds keys and values per key-value head: model B stores 8 sets in each layer to model A's 32. Each request on model A holds four times the cache.

</details>

<details><summary>B. Model B, one or two more requests</summary>

✗ Fewer key-value heads read as a small trim to the model, and a trim to the model buys one or two more requests at most. What shrinks is the cache, which every token of every request adds to: model B's is one fourth the size of model A's, and with the weights leaving most of the GPU free, that cache decides how many requests fit.

</details>

<details><summary>C. Model B, several times as many</summary>

✓ Right. Model B holds about four times as many requests as model A on the same GPU. Its cache per token is one fourth the size of model A's, while the weights of the two take the same space.

Pick model A because the two sit side by side on a leaderboard, and the same GPU serves about a quarter as many users at once, a difference that no column of the leaderboard shows and the hardware budget pays for every month.

In each layer, every token leaves one key and one value per key-value head in the cache. Model A gives each of its 32 attention heads a key-value head of its own; in model B, groups of four attention heads share one, so it stores 8 sets in each layer to model A's 32. Picture an 80 GB GPU, with each head's key and value 128 numbers long: the weights take about 14 GB either way, model A's cache takes 512 KiB per token and model B's 128 KiB, so a 4,000-token request holds about 2.1 GB on model A and 0.5 GB on model B, and the free memory fits about 31 requests against about 125. Before you size a model's cache, read its key-value heads in the configuration; the parameter count hides them.

</details>

---

###### 6. Raising the context length past what the model card states

A startup answers questions about long engineering standards with an open-weight model on its own server. The model's card says it supports 32,000 tokens. You set the server's context length to 128,000 and allow it to go past the model's stated length. You change no other setting of the model or the server. A 90,000-token standard comes in.

What should you expect from the answers about that standard?

<details><summary>A. Answers as good as on a short standard</summary>

✗ On your own server the context length is a number you type, and the server took 128,000, so the limit appears to be yours to set. The setting decides how many tokens the server accepts. What the model can use was fixed in training, at 32,000, and the 58,000 tokens beyond it sit at positions it never learned.

</details>

<details><summary>B. Answers that become unreliable</summary>

✓ Right. Past the length the model was trained for, its answers become unreliable and can turn to nonsense. The server takes all 90,000 tokens, and the model has never learned what to do with positions beyond 32,000.

An engineer asks for the test limits in the standard and gets a confident list in which some values were never in it, while the request itself comes back like any other.

A model's context length is set by how it was built and trained: its weights learned how tokens at positions up to 32,000 relate to each other, and nothing taught them the positions beyond. The server setting decides how many tokens the server takes in, not what the model can use, so the whole standard goes in, and every token past 32,000 sits at a position the weights never saw. Each later token attends across those positions, so the damage spreads through the answer. A 2021 study, Train Short, Test Long, found that models using the common ways of marking position did poorly on texts longer than they were trained on; those were 2021 models, so measure yours. On a model of an older design, which learned one fixed entry for each position up to its stated length, a server can fail the request with an error instead. To read longer standards, pick a model whose card states the length you need, turn on an option for longer texts if the card documents one, or split the standard into parts that each fit.

</details>

<details><summary>C. Answers drawn from the first 32,000 tokens alone</summary>

✗ A reader short of time gets through as much of a long file as they can and stops there. Nothing in the model stops at its stated length. Every token goes in, and each later token's attention reaches back across positions the model never saw in training, so the trouble spreads through the answer instead of ending at a line.

</details>

<details><summary>D. Answers drawn from the last 32,000 tokens alone</summary>

✗ The chat apps you use keep the newest part of a long conversation and drop the oldest once it runs long. That dropping is work an app does before the request is sent. Here the server sends all 90,000 tokens, and the model has no way to set the earliest aside; it reads all of them, positions it never learned included.

</details>

---

###### 7. Moving a chat on your own server to 4-bit weights

The support chat of a travel agency runs on a server the agency runs itself, and by evening the GPU is full of long chats. To fit more of them, you switch to a 4-bit version of the same model and leave every other server setting at its default.

What does the switch shrink?

<details><summary>A. The weights alone</summary>

✓ Right. The weights take about a quarter of their old memory, and each chat's key-value cache takes as much as before. With default settings the cache keeps the precision it had, usually 16 bits, whatever the weights are stored in.

The support lead expected the 4-bit model to clear the evening queue. It fits some more chats, nowhere near four times as many, and customers still wait for a reply at eight in the evening.

Quantization shrinks only the numbers it is applied to, and a 4-bit model is one whose weights are stored in 4 bits. The key-value cache is computed while the model runs, and servers keep it at its default, usually 16 bits, unless you change its own setting, so each long chat costs what it did. Picture 16 GB of weights: at 4 bits they take about 4 GB, freeing 12 GB, while a 30,000-token chat, at 128 KiB of cache per token, still holds about 3.9 GB, so the freed room holds about three more such chats. Storing the cache in 8 or 4 bits is a separate server setting, which halves or quarters it at a cost in exactness, and one server's docs warn that the 4-bit cache's loss shows more at long context lengths. Test your longest chats before you quantize the cache.

</details>

<details><summary>B. The weights and each chat's key-value cache</summary>

✗ A 4-bit model, by its name, is 4-bit throughout, cache included. The label describes how the weights are stored, and nothing else. The key-value cache is computed while the model runs, and with default settings servers keep it as it was, usually at 16 bits, so each long chat's cache is exactly the size it was before the switch.

</details>

<details><summary>C. The model file on disk alone</summary>

✗ A 4-bit file can pass for a zipped download that is unpacked to full size once loaded. Quantized weights stay compressed in memory, and that is what lets a model too large for a GPU at 16 bits run on one at 4. The weights take about a quarter of their old room; only the cache keeps its size.

</details>

---

###### 8. Adding a draft model to a bank's internal assistant

Staff at a bank use an internal assistant that runs a large model on your own server. In quiet hours a reply still feels slow to the one or two people using it, so you turn on speculative decoding, choosing a small model from the same family as the draft model, and replies arrive faster. A compliance officer asks whether staff now read text partly written by the small model.

Compared with the replies the large model wrote alone, what do staff read now?

<details><summary>A. Text partly the small model's, a little weaker</summary>

✗ When a second writer joins, you expect their voice to show, and a smaller model writes worse. The small model only proposes. A proposed token survives only by the large model's own probabilities, and any it rejects is replaced by a draw from the large model, so no token reaches the reply on the small model's say.

</details>

<details><summary>B. The small model's phrasing, fact-checked by the large one</summary>

✗ A human editor checks the facts and leaves a junior writer's phrasing alone, and the large model could be pictured working that way. It does not judge facts apart from words: it weighs every token, wording included, against what it would have drawn itself, and the text that comes out follows its probabilities word by word.

</details>

<details><summary>C. Better replies than the large model's alone</summary>

✗ In code review a second reader catches what the first one missed, and you would hope for that from a second model. The draft model never overrules the large one; it only proposes tokens, which the large model accepts or replaces. The replies come out as the large model would write them, no better and no worse.

</details>

<details><summary>D. What the large model alone would write</summary>

✓ Right. The replies come out as the large model alone would write them, drawn from the same probabilities; only the speed changed. Every token a reader sees passed the large model's own check.

The compliance officer can rely on the evaluation the team ran on the large model alone, since it still describes what staff read. What needs watching is the speed at busy hours: the gain is meant for quiet ones, when each pass has room to spare.

Each writing pass reads the large model's weights from memory to produce one token. Speculative decoding has the small draft model guess the next few tokens, and the large model checks all of them in one pass, the way it reads a prompt: each guessed position looks only at the tokens before it, so all of them can be scored together. The large model keeps each guess with a chance set by comparing its own probability for that token with the draft model's, and always keeps it when its own is at least as high. At the first guess it turns down, it draws a replacement from what remains of its own probabilities once the draft's are subtracted, so the reply follows the large model's probabilities exactly. A poor draft model costs speed, never wording. The 2022 paper that introduced the method showed identical outputs and a two- to threefold speed-up on its models, and one server's docs add that the hardware's rounding of numbers can shift the probabilities very slightly.

</details>

---

###### 9. Reading an attention map to explain a fraud flag

An online electronics shop flags risky orders with a model it hosts itself, run through a library that returns the attention weights. An order from a new account was flagged, and a fraud analyst asks why. You take the last layer's attention weights and average them over its heads. The map shows the share that the request's final token, the one the model writes the flag from, gives to each part of the order, listed as they appear in the request:

```text
account age        0.03
order lines        0.41
delivery address   0.33
payment note       0.23
```

What can you tell the analyst about account age from this map?

<details><summary>A. That account age barely counted toward the flag</summary>

✗ A thin slice of a pie chart is a small share of the whole, and 0.03 reads the same way. The weight is where the final token looked in this one layer. Earlier layers can already have mixed what account age says into the tokens after it, and the final token can take it from them.

</details>

<details><summary>B. That account age counted least of the four parts</summary>

✗ Even if the numbers are off, their order is the part you would put in a report. It is still the order of where one layer looked. The parts ranked high can carry account age's information inside them, and a small share of a value that carries the deciding signal can outweigh a large share of one that carries little.

</details>

<details><summary>C. Nothing about how much account age counted</summary>

✓ Right. The map shows where the final token looked in the last layer, not how much account age moved the flag. By that layer, the parts that come after account age can already carry what it said.

The analyst's case file says account age played no part, and the shop's next review of its fraud rules drops the one signal that may have tipped this order, on the strength of a number that never measured it.

Attention weights are the shares one token gives to the tokens before it, in one head of one layer, and every layer passes its mix on to the next. In the earlier layers, the order lines, the address and the note, which all come after account age, can take what it says into their own values; by the last layer the final token can get it from them while giving account age itself 0.03. A weight also says nothing about how much what it brings changes the result: a small share of a value that carries the deciding signal can count for more than a large share of one that carries little. A 2019 study, Attention is not Explanation, tested the text classifiers of its time. It found that attention weights often disagreed with other measures of which inputs mattered, and that quite different weights could give the same prediction; those were 2019 models, so treat it as a warning about yours. To learn whether account age matters, change it and resend the same orders: if the flags move, it does.

</details>

<details><summary>D. That every layer's map, added up, would show how much it counted</summary>

✗ When one layer is not enough, adding up all of them is the natural repair. Every layer's map shares the limit of this one: it shows shares of attention, not what the information they carry did to the result. A combined map shows where information could have flowed; changing the account age and resending shows whether it moved the flag.

</details>

---

## Further reading

- [GQA](https://arxiv.org/abs/2305.13245), 2023: why writing each token waits on loading the keys and values, and how letting attention heads share them shrinks the cache.
- [Train Short, Test Long](https://arxiv.org/abs/2108.12409), 2021: how models handle texts longer than those they were trained on, and a way of encoding position that does better.
- [Fast Inference from Transformers via Speculative Decoding](https://arxiv.org/abs/2211.17192), 2022: how a small model's guesses, checked by the large model in one pass, speed up writing without changing the outputs.
- [Attention is not Explanation](https://arxiv.org/abs/1902.10186), 2019: tests on text classifiers showing that attention weights often disagree with other measures of which inputs mattered.

Next: [Generation parameters](../generation-parameters/README.md)
