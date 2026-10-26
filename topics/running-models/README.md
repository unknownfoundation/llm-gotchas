# Running models

Needs: [Inside the model](../inside-the-model/README.md) · [Generation parameters](../generation-parameters/README.md) · Reviewed October 2026

Running a model means keeping its weights loaded on machines that take requests and send back replies. One model can run in four kinds of place: on its maker's own machines, inside a cloud company's platform that sells models from many makers, at a hosting service for open-weight models, or on hardware you run yourself. The place decides who handles your requests, which limits and rules they meet, and what you have to set up yourself.

- **Token**: a piece of text, often a word or part of one; requests, limits and prices are counted in tokens, and each model's tokenizer splits text into the tokens of its own vocabulary.
- **Weights**: the numbers a model learned in training; running it means loading them into memory and computing with them.
- **Open-weight model**: a model whose maker publishes its weights for anyone to download and run; a closed model runs only where its maker offers it.
- **Region**: one of a cloud platform's areas of data centres, such as a city or a country; you choose the regions your account and its resources sit in.
- **Quota**: the cloud platforms' name for rate limits, the number of tokens and requests your account may send to a model each minute.
- **Finish reason**: the field in a response that says why the reply ended, such as a natural end or the generation cap.
- **Reserved capacity**: a set amount of a model's throughput, the requests and tokens it can handle each minute, that a platform holds for your account alone, instead of sharing its machines among all its customers; often called provisioned throughput.
- **Chat template**: the format, shipped with a model's files, that turns a list of messages into the one text the model reads, marking where each turn, one message of the chat, starts and ends, usually with special tokens.
- **Special token**: a token reserved for structure, such as the start of a turn, with a marker string and an ID of its own in the model's vocabulary.
- **Chat endpoint and text endpoint**: the two usual ways into a server you run. The chat endpoint takes a list of messages and fills in the model's chat template. The text endpoint takes one finished string: most servers pass it to the model as it is, and some apply the template to it unless you switch that off.
- **Context length**: the size of the context window, the largest number of tokens a single request can carry; on your own server it is a setting, and it can be lower than the length the model's maker publishes.
- **Key-value cache**: the memory that holds a request's work on its tokens while the request runs, growing with its length; a server needs room for it beside the weights.

Whichever company runs the model behind a provider's API, its maker, a cloud platform or a hosting service, sets the limits, the filters and the prices; on your own server you load an open-weight model's files on hardware you own or rent, and the prompt format, the defaults, the context length and the memory are yours to set. A model turns a request into a reply the same way wherever it runs.

A model can run in four places:

| Place | Who runs the model | Which models | Kind of system |
| --- | --- | --- | --- |
| The maker's API | the company that made the model | that maker's own models | a provider's API |
| A cloud platform | a cloud company, as a service in the same cloud as your account | many makers' models, closed ones included | a provider's API |
| A hosting service | a company that runs open-weight models others made | open-weight models | a provider's API |
| Your own server | you, on hardware you own or rent | open-weight models | your own server |

A router is not a fifth place: it is one API in front of several providers, and it passes each request to one of them, so the same model name can reach hosts that run it differently, for example with the weights stored at lower precision, in fewer bits each. The same model on a cloud platform can lack some of the maker's features, and it comes with the platform's own request format, model names and quotas.

Where a model runs bites when code moves to a new place or onto a new model, and when an auditor asks where the requests went.

This topic lets you:

- choose where a model runs when a tool must stay the same for years, or when an auditor will ask where every request was processed
- move a feature onto a cloud platform with its quotas, its content filter and a week of reserved capacity planned for
- change the model behind your own server, or give part of its card to another service, and know what to check first

###### 1. Choosing where a hospital's report sorter runs for five years

A hospital's radiology team builds a tool that sorts incoming reports by urgency. The regulator will approve the tool together with one exact model, chosen now, and any change to that model means a new review lasting months. The tool has to keep running unchanged for five years.

Which plan keeps the chosen model answering for all five years?

<details><summary>A. A version name with its release date, on the maker's API</summary>

✗ Versions on a provider's API carry dates for exactly this reason, so that code can ask for one release and get it. The date fixes which release answers; it does not keep the release on offer. Makers retire old versions on their own schedule, after which a request naming one fails, and nothing stops that date from falling inside the five years.

</details>

<details><summary>B. That same version name, called through a cloud platform</summary>

✗ The hospital already holds a contract with its cloud that runs for years, and a model bought through that contract feels covered by it. The platform sets its own retirement dates for the models it serves, apart from the maker's, and they are no more under the hospital's control. A dated name there fixes the release, not how long it stays.

</details>

<details><summary>C. An open-weight model through a hosting service's API</summary>

✗ Published weights cannot be taken back, so an open-weight model seems safe from retirement wherever it is served. The weights stay published, but a hosting service chooses which models it keeps serving and can stop serving one on a date it sets. The permanence belongs to the files, and only a copy you hold gives you that.

</details>

<details><summary>D. An open-weight model on the hospital's own server</summary>

✓ Right. Only the copy the hospital keeps stays the same: the weights are files on its own disks, and the model changes only when someone there replaces them.

A plan that rests on another company's machines works until a notice arrives. The version is retired on a date the hospital did not choose, and the radiologists' tool must either stop or go back to the regulator with a new model, a review of months landing somewhere in the middle of the five years.

Where a model runs decides who can take it away. On a provider's API you get the versions its operator chooses to keep: makers retire old versions on a published schedule, and requests to a retired version fail; a cloud platform sets its own retirement dates for the models it serves, which can differ from the maker's; a hosting service can stop serving a model on a date it sets. A dated name fixes which release you call, not how long it is offered.

An open-weight model is the one kind whose weights you can hold, and on your own server the weights, the chat template and the server's settings are files and settings you control, so the same model runs for as long as you keep them. Keep a second copy of the files away from the server, and keep the server software at the same version too, since an upgrade can change how the same weights are run.

</details>

---

###### 2. Setting up a claims assistant in a Zurich region

An insurer in Switzerland moves its claims assistant onto a cloud platform. You set up the account and every resource in the platform's Zurich region, whose page says stored data stays in Switzerland. The platform offers several ways to call the model, and you pick the one its guide recommends first, which has the highest limits and the newest models. Months later, an auditor asks where the model processed the claims.

Where can a claim have been processed?

<details><summary>A. Only in the Zurich region, where the account is set up</summary>

✗ Every screen of the console says Zurich, from the account to the stored files. Stored data and processing follow separate rules. The recommended option sends each request to whichever region has capacity, and that region can be in another country; only an option tied to one region keeps the work in Zurich.

</details>

<details><summary>B. In whichever region had room, abroad included</summary>

✓ Right. With the recommended option, each request went to whichever of the platform's regions had capacity at that moment, and that can be a region in another country. The stored data stayed in Switzerland; the processing did not have to.

The insurer promises its clients that claims are handled in Switzerland. The auditor now holds an answer that breaks the promise for an unknown share of several months of claims, and none of those requests can be sent back.

A region is where your account and its stored data sit, and a request can still be processed somewhere else. To serve more traffic and newer models, cloud platforms offer ways to call a model that send each request to any region with room, either worldwide or within one area such as a group of countries. These are the options their guides recommend first: they carry the highest limits and the newest models, and often the lowest price. An option tied to one area or one region keeps the processing inside it, at a higher price or with lower limits, and the newest models may not be offered there at all. Where data must stay in a country, choose the option by where it processes requests, not by where the account sits, and check that the model you need is served there. Some platforms log the region that handled each request, which is the record to read when the question comes.

</details>

<details><summary>C. In any data centre in Switzerland, Zurich or elsewhere</summary>

✗ Spreading load over several data centres is how a service stays up, and with the account in a Swiss region the Swiss border is the natural edge to picture. The recommended option has no national boundary: it can send a request to any region the platform runs. Options bounded by one area or one region exist, and you have to choose them.

</details>

<details><summary>D. In the region closest to the customer who sent it</summary>

✗ A content network answers each visitor from the nearest point, and a platform serving millions could seem to work the same way. The model option routes by where capacity is free, not by where the customer sits, so a claim typed in Basel can be processed far from both Basel and Zurich.

</details>

---

###### 3. Opening a second region before a sales week

A shop that sells sports gear runs its support bot through a cloud platform, in one region. Last month you had the model's quota raised for the sales week ahead. Now you open a second region with the same model and move the traffic of shoppers in Europe there.

In the busiest hour of the sale, how much traffic will the second region accept?

<details><summary>A. Part of last month's raised quota, which both regions draw on</summary>

✗ On most APIs you have used, the rate limit belongs to the whole account. On a cloud platform the quota is kept per region and per model. The second region does not draw on the first region's raise; it starts from its own default.

</details>

<details><summary>B. A second full copy of last month's raised quota</summary>

✗ The raise was approved for this very model, and an approval usually travels with the thing it names. It applied to one region's quota for that model. A new region needs a request of its own, and until one is approved it serves at its default.

</details>

<details><summary>C. Its own quota, at the platform's default level</summary>

✓ Right. The second region has a quota of its own for that model, at the default the platform gives your account there, and last month's raise stays in the first region.

On the first morning of the sale, shoppers in Europe get error pages in the busy hours while the first region has room to spare. A fresh request for more quota is reviewed by hand and can take days, so the sale may be over before it is approved.

A quota on a cloud platform is a rate limit kept per model and per region, often per way of calling it as well, and each one is raised by its own request. Opening a region, or calling a model you have not used there before, starts from that place's default, which can be lower than the published figure. Before you move traffic, request the quota in the new region for the model you will call there, early enough for the review, and read each rate-limit error before you act on it, since some platforms return the same error when their machines are busy rather than when your quota is spent.

</details>

<details><summary>D. As much as the region's machines can serve</summary>

✗ On a cloud, the services you run yourself scale out as traffic grows, and a model served by the same cloud could do the same. Model calls meet a quota before any of that, and past it the platform refuses each request with a rate-limit error, however many machines the region has free.

</details>

---

###### 4. A scary chapter in a children's story app

A children's story app writes each next page with a model on a cloud platform, and the platform's content filter is switched on for that model. Your code waits for the whole reply to arrive in one response, rather than piece by piece as it is written. It shows an error screen when a call fails and otherwise saves the reply as the next page. A child asks for a scary chapter about a dragon. The filter lets the request through, then blocks the draft the model writes.

What does the call return for that draft?

<details><summary>A. A normal response whose finish reason names the filter</summary>

✓ Right. The call succeeds, the draft does not come back, and the finish reason names the filter. Your code saves whatever text it carries as the page. That text depends on the platform: nothing at all, a notice you set up on the platform, or, on a rare long reply, the part of the draft written before the block.

At bedtime a child turns to the dragon chapter and finds a blank page, or a line about content that can't be shown, saved into the book as if the story had written it. The error screen never appeared, so the app's logs count the call as a success.

A content filter checks the prompt and the reply outside the model. When it blocks a reply, the platform still answers the request: the response arrives as a success and carries a finish reason that names the filter where a natural end would be. One platform returns no text in that case, or on a rare long reply the part written before the block; another puts in the message you set up for blocked replies. A prompt the filter blocks can be handled differently, as an error on one platform and as the same kind of success on another. Check the finish reason on every response before you use its text, and treat a filter's finish reason as no answer: try again with a gentler request, or tell the reader the chapter could not be written.

</details>

<details><summary>B. A failed call, and the app's error screen</summary>

✗ From where your app sits, a blocked chapter is a failure, and failures usually arrive as errors. The platform treats a blocked reply as an answer: the request returns normally, so the error screen never shows and the save runs on whatever the response holds.

</details>

<details><summary>C. The draft with the scary lines cut out</summary>

✗ In-game chat filters star out one word and let the rest of a message through. This filter does not edit the draft line by line. It blocks the reply, and the response carries a finish reason naming the filter where the draft would have been.

</details>

<details><summary>D. A refusal the model wrote, with a natural end as the finish reason</summary>

✗ Models do turn down requests in their own words, and a polite refusal may be what you met in testing. Here the draft was written in full and then stopped by the filter, outside the model. No refusal from the model reaches the response; what arrives names the filter as the reason the reply ended.

</details>

---

###### 5. Reserving capacity for an airline's holiday week

An airline's rebooking assistant gets its replies from a model on a cloud platform. For the holiday week you buy reserved capacity for the model, sized to the forecast for the busiest hours, and you delete it when the week ends. The storms behind the forecast never come, and traffic runs at half of it all week.

What does the week's reserved capacity cost?

<details><summary>A. A bill per token served, at a discount for reserving</summary>

✗ A train ticket bought a week ahead costs less, and capacity reserved a week ahead reads as the same deal: a lower price for each token. Reserved capacity is billed by the hour for what is held, not by the token. A quiet hour costs the same as a busy one.

</details>

<details><summary>B. The per-token bill you would pay without reserving</summary>

✗ A reservation can sound like a promise of room, with the bill left as it was. The hourly charge for holding the capacity is the bill: the platform does not count your tokens against it, and the quiet hours are charged in full.

</details>

<details><summary>C. A booking fee, with nothing for idle hours</summary>

✗ A restaurant holding a table asks for a deposit at most. The platform keeps those machines from every other customer for the whole week, and it charges for every hour it does so, used or not.

</details>

<details><summary>D. The full amount you reserved</summary>

✓ Right. The bill counts every hour the capacity was held, 168 hours for the week, whatever share of it the traffic used. At half the forecast, each token served cost twice what the plan priced it at.

Finance sets the holiday week beside the plan and finds the full reservation billed for half the passengers, with no line that credits the quiet hours, then asks why the airline paid for a storm that never came.

Reserved capacity is machines held for your account alone, so the platform bills for holding them: an hourly rate for the amount reserved, from the moment it is set up until it is deleted, however few tokens pass through. That is the trade against paying per token, where an idle hour costs nothing. Steady, heavy traffic comes out cheaper reserved, and a forecast that misses leaves you paying for the gap. Some platforms sell it with a commitment of a month or more at a lower hourly rate, and then it cannot be deleted before the term ends. Reserve for the traffic you are sure of and let the rest go per token; where the platform offers it, requests beyond the reservation can spill over to the platform's ordinary per-token service.

</details>

---

###### 6. Swapping the model behind a bike shop's repair chat

A bike shop answers repair questions through a chat that runs on your own server. Your code writes the prompt string by hand, wrapping each turn in the markers the old model's chat template uses. It sends that string through `call_model` to the server's text endpoint, which is set to apply no template. This week the server switches to a model from a different family, and the code stays as it is. The team chose the new model in a side-by-side trial that went through the server's chat endpoint.

```python
def build_prompt(question):
    return ("<|im_start|>system\nYou answer questions for a bike repair shop.<|im_end|>\n"
            "<|im_start|>user\n" + question + "<|im_end|>\n"
            "<|im_start|>assistant\n")

reply = call_model(prompt=build_prompt("How often should I replace my chain?"))
```

What does the new model read where your string says `<|im_start|>`?

<details><summary>A. The same turn marker the old model read, shared by all chat models</summary>

✗ You have seen this layout in example after example, all of them written for the old family. The layout belongs to that family alone, and its markers are special tokens in its vocabulary. The new family was trained on chats wrapped in other markers, and your string carries none of them.

</details>

<details><summary>B. Nothing: the tokenizer rejects the unknown marker with an error</summary>

✗ Compilers and JSON parsers refuse input they cannot read, and a tokenizer sits in the same place in the pipeline. This one has no input it cannot read: it can build any text from pieces as small as one byte. The request goes through, and the model answers.

</details>

<details><summary>C. Its characters as ordinary text, split into several tokens</summary>

✓ Right. The new model reads your markers as ordinary text broken into fragments, and not one of the turn markers it learned in training. Its tokenizer has no token for `<|im_start|>`, so each occurrence becomes six ordinary tokens: `<`, `|`, `im`, `_start`, `|` and `>`, and the prompt that was 30 tokens for the old model is 52 for the new one.

The call succeeds and replies come back, so the swap looks done. Customers asking about chains and brakes get answers from a model reading a layout it never saw in training, and the trial that picked the new model went through the chat endpoint, so it tried a prompt your chat never sends.

A chat template belongs to one model family: it wraps each turn in that family's markers, and those markers are special tokens in that family's vocabulary only. The old template's strings mean nothing special to the new tokenizer, which spells them out of ordinary pieces like any other text, and the new model's own markers, special tokens that open each turn with its role and close it at the end, appear nowhere in your string. Nothing fails, because the new tokenizer can spell out any text from pieces as small as one byte. Send the chat to the chat endpoint as a list of messages instead: the server fills in the chat template from the loaded model's own files, so a swap brings its format along. On a provider's chat request your code sends roles as fields and the provider writes the markers; the mismatch comes from writing the string by hand.

</details>

<details><summary>D. Its own family's turn marker, swapped in by the tokenizer</summary>

✗ The new model's tokenizer knows every one of its own markers, which makes it the obvious place to translate yours. It only splits text into tokens, and a string it has no token for becomes ordinary pieces. Putting the right markers in is what a chat template does, and this text endpoint applies none.

</details>

---

###### 7. Starting a second service on the chat server's GPU

A design studio serves its chat model from its own server, on one GPU: a graphics card with 48 GB of memory, which the server shares with nothing else. The model's weights fill a third of that memory. The server is set to work on up to 16 requests at once, each up to 12,000 tokens long. It is up with the model loaded, and before the first chat of the morning a colleague plans to start a search service that needs 8 GB on the same card.

Before the first chat request, what takes up the card's memory?

<details><summary>A. The weights alone, until requests arrive</summary>

✗ A desktop program takes memory as it needs it and hands it back, and an idle server has nothing to hold. A server set to work on many requests at once plans for its busiest moment instead: when it loads the model, it sets aside room for the caches of the requests it is set to serve, so that memory is held with no chat running.

</details>

<details><summary>B. The weights and a large reserve for requests to come</summary>

✓ Right. Once it has loaded the model, the server holds the weights and a large block of memory for the key-value caches of the requests it is set to serve, before any request arrives. The colleague finds far less free than the 32 GB the weights leave.

The search service fails to start on a card where no chat is running, and whoever opens the GPU's memory readout finds an idle server holding most of it, which passes for a leak until someone reads the server's settings.

A server works on many requests in one pass, and each request needs a key-value cache that grows with its tokens, so the server sets that memory aside when it loads the model instead of asking for it in the middle of a request, when the card might be full. How much it takes depends on the server. Some size the block from their settings, the number of requests at once and their length; others take a fixed share of the card and fill what the weights leave with cache. Picture each token's cache taking 128 KiB of memory: 16 requests of 12,000 tokens at that size set aside about 25.2 GB, and with the 16 GB of weights the server holds 41.2 GB, leaving 6.8 GB. A server set to take 92% of the card leaves 3.84 GB. Either way the 8 GB service does not fit. To share a card, set the server's share of memory, or its number of requests and their length, so that the other service's 8 GB stays free.

</details>

<details><summary>C. The weights and room for a single 12,000-token request only</summary>

✗ One full-length request is the biggest single thing a server must fit, and a reserve of exactly that is what a careful engineer would set. This server is set to work on 16 requests at once, and it sets aside room for all of them, or takes a fixed share of the card for as many as fit.

</details>

<details><summary>D. One copy of the weights for each of the 16 requests</summary>

✗ Sixteen users at once brings to mind sixteen copies, the way sixteen people each open their own program. The weights go onto the card a single time, and every request passes through that one copy in the same pass. What each request adds is its own key-value cache, not a copy of the model.

</details>

---

## Further reading

- [Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/abs/2309.06180), 2023: why the memory each request needs is hard to manage on a busy server, and a way of handing it out in small pages with near-zero waste.
- [Quantifying Language Models' Sensitivity to Spurious Features in Prompt Design](https://arxiv.org/abs/2310.11324), 2023: how well a prompt's format suited one model was only a weak guide to how well it suited another, which matters whenever a prompt moves to a new model.

Next: [Data and privacy](../data-and-privacy/README.md)
