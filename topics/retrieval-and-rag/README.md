# Retrieval and RAG

Needs: [Context window](../context-window/README.md) · [Embeddings and vector search](../embeddings-and-vector-search/README.md) · Reviewed October 2026

Retrieval-augmented generation, or RAG, answers questions from documents the model never saw in training. Before each call, your code searches those documents for the text that best matches the question and pastes that text into the request beside the question; the model reads that request and nothing else of your documents. The work runs in five steps: split each document into chunks, embed each chunk for a vector search, store the chunks in an index, search the index with a query for each question, and paste what the search returns into the request.

- **Embedding**: a list of numbers, also called a vector, that an embedding model computes from a text, so that texts with similar meaning get similar lists. To embed a text is to compute its embedding.
- **Chunk**: a piece of a document, cut by a splitter to a size you choose, sometimes with an overlap that repeats the end of one chunk at the start of the next. Chunks are what gets embedded, stored, found and pasted.
- **Index**: the store your search runs over, holding each chunk under an id with its text, its embedding and metadata such as its source document, its department or the customer it belongs to. Some stores call one index a collection.
- **Top-k**: the chunks a search ranks highest, best first, each returned with its score; k is the number you ask for. A generation setting with the same name is a different thing: it limits the model's choice of each next token to the k likeliest.
- **Query**: the text your code hands to the search.
- **Upsert**: a write whose name joins update and insert, one call that adds records to the index and updates records already in it.
- **Metadata filter**: a condition on the metadata, such as one department or one customer, that you pass to the search with the query, so it returns only chunks that meet it.
- **Grounded reply**: a reply whose every claim comes from the chunks in its request.
- **Reranker**: a second model that reads the query and one chunk together and scores how well the chunk answers it, so the chunks can be put in a better order.

With a provider's API, your code can run every step itself. Where the provider offers one, you can instead upload files to a built-in file search, which splits, embeds and searches them and passes what it finds to the model. On your own server, every step from splitting to search is code you run next to the model. Retrieval works the same wherever the model runs.

A search can find chunks in three ways:

| Way | What it does |
| --- | --- |
| Keyword search | scores a chunk by the words it shares with the query, also called lexical search; BM25 is the standard formula, weighing each shared word by how often it appears in the chunk and how rare it is across the chunks |
| Vector search | embeds the query and returns the chunks whose embeddings sit nearest to it, usually by cosine similarity, a score that grows as two embeddings point more nearly the same way |
| Hybrid search | runs both on the same query and merges the two result lists into one |

Retrieval bites between the steps. Each step has its own settings, and a wrong setting seldom raises an error: it changes which text reaches the model.

This topic lets you:

- work out what a request carries after a split and a search, when the documents hold no answer, when one section repeats across many of them, or when a user follows up on a reply
- keep an index current and separate for each business and department, and judge what a cited or pasted chunk proves about an answer
- choose where a reranker runs and how keyword and vector results are merged, so the chunk that answers is among the few a request has room for

###### 1. Splitting a shop's returns policy for the index

Till staff at a chain of clothing shops check returns with an assistant that answers from the store policies. Your indexer cuts them into chunks of 200 tokens with no overlap, and each question gets the top three. In the returns policy, chunk 7 ends with "Sale items can be returned within 30 days." and chunk 8 opens with "Swimwear is the exception: it cannot be returned once worn." Both sentences sit in the Returns section, and every section of the policy runs under 400 tokens. A question about a sale swimsuit gets chunk 7 and not chunk 8.

Which change makes sure the exception reaches the model whenever the rule does?

<details><summary>A. Cut chunks of 100 tokens instead</summary>

✗ Smaller chunks promise sharper hits, each one closer to a single point. They also add cuts, and every boundary at 200 tokens is still a boundary at 100, since 100 divides 200. The rule still closes one chunk and the exception still opens the next, so a search that finds the rule can still send it alone.

</details>

<details><summary>B. Add an overlap of 50 tokens between chunks</summary>

✗ Overlap is the standard advice for a fact that straddles a cut, and here it half works. Each chunk now repeats the last 50 tokens of the one before, so one chunk carries the rule and the exception together. The cuts still fall wherever the count runs out, and whenever one lands between the two sentences, the chunk that ends there sends the rule alone, which the search can rank first.

</details>

<details><summary>C. Send the top five chunks instead of three</summary>

✗ More hits feel like a wider net, and chunk 8 is the very next piece of the document. Its place in the results is another matter: the search ranks chunk 8 by its own words, and nothing ties it to chunk 7. It comes along only when its own words earn it one of the five places.

</details>

<details><summary>D. Cut at the policy's headings, one section per chunk</summary>

✓ Right. Cut at the headings, and the Returns section becomes one chunk with the rule and its exception inside it. Whenever the search finds the rule, the request carries the exception too.

With the 200-token cuts, a customer brings a worn swimsuit from the sale back to the till, the cashier asks the assistant, and the request holds the 30-day rule with no word about swimwear. The refund goes through, and the store manager finds the worn swimsuit in the returns bin at closing.

A chunk is all the model sees of its document, and the splitter decides what travels together. A cut by size falls wherever the count runs out, here between a rule and its exception. Smaller chunks only add cuts. An overlap copies the end of each chunk into the start of the next, so one chunk holds both sentences, while any chunk that ends between them still sends the rule alone. More results bring chunk 8 only when it ranks on its own. Splitting at the document's headings keeps each section whole, as long as a section fits the embedding model's input limit; cut a longer one at its paragraphs and repeat the heading on each part. Another way keeps each chunk's position in its document and pastes its neighbours in with it.

</details>

---

###### 2. A question the manuals do not answer

You run the help bot of a bike shop. Its index holds chunks of the shop's product manuals and nothing else, and your code pastes the top five chunks for each question into the request. On a Saturday evening a customer asks whether the shop is open on Sunday. No manual mentions opening hours.

Which chunks does the request for the Sunday question carry?

<details><summary>A. No chunks, only the customer's question</summary>

✗ A database query whose condition no row meets returns nothing, and a search over an index feels like the same kind of lookup. A vector search has no such condition: every chunk has a distance to the query, however large. It ranks them all and hands back the five nearest, and your code pastes all five above a question about Sunday.

</details>

<details><summary>B. Five chunks from the manuals</summary>

✓ Right. The request carries five chunks from the manuals, the five closest to a question about Sunday, about batteries, brakes or whatever else lands nearest. They sit in the request exactly where an answering chunk would, above the customer's question.

The customer's question about Sunday arrives in a request built around brake pads and battery care, with nothing in it about opening hours. If the reply guesses, the customer cycles over on Sunday and finds the shutters down.

Top-k is a ranking: the search orders every chunk by its distance to the query and hands back the first k, with no test of whether any of them answers. A question no chunk answers still gets five. Read the score that comes back with each hit, drop the hits past a line you chose on your own data, and when nothing is left, reply that the manuals do not cover it without calling the model.

</details>

<details><summary>C. Only the one or two chunks that match the question well</summary>

✗ Each hit carries a score, and it is natural to read a score as a pass mark the store applies. The store returns the scores and applies no cutoff of its own, so a search for five returns five. A cutoff exists only where your code reads the scores and drops the far hits, and this code pastes every hit.

</details>

---

###### 3. Raising k on a train conductors' assistant

A train operator gives its conductors an assistant that answers from 40 regional timetables and one fare table, which lists a bike ticket. Each timetable carries the same bike section, pasted in word for word, and your splitter makes every section a chunk of its own. For "Can a passenger bring a bike, and does it need a ticket?" the replies cover the bike rules and never mention the bike ticket, so you raise k from 5 to 10. Assume the bike section is the chunk nearest to that question.

What does the request for that question carry now?

<details><summary>A. Ten copies of the bike section</summary>

✓ Right. Ten copies of the bike section fill the request, one from each of ten timetables, and nothing from the fare table comes with them.

A cyclist boards at a country station without a bike ticket, the conductor checks the assistant, and the reply talks about folding bikes and rush hours and says nothing about a fare. Every question about bikes now also sends twice the text it did, all of it one paragraph.

Top-k ranks each chunk on its own and never checks whether two of them say the same thing. Forty identical chunks get forty identical embeddings and forty identical scores, so they hold the first 40 places, and any other chunk starts at 41st. Raising k to 10, or to 40, sends more copies. Remove the repeats before they take the places: index a shared section once, with the timetables it belongs to listed in its metadata, or keep one chunk per distinct text when you take the top results, comparing each text's hash, a short code computed from its exact characters. Picture these documents with the repeats removed: the bike ticket then comes second. Near copies that differ by a word score almost alike and crowd the list the same way, so a step that skips any result too close to one already picked spreads the places across different texts.

</details>

<details><summary>B. The bike section once, then nine other chunks with the bike ticket among them</summary>

✗ To a conductor, forty copies of one paragraph are one fact, and you expect the search to treat them as one. The index keeps each copy under its own id and scores every one of them, and identical text gets an identical score. With the bike section nearest, its 40 copies take the first 40 places, and the bike ticket comes after them.

</details>

<details><summary>C. The bike section five times, then five other chunks with the bike ticket among them</summary>

✗ Doubling the places for a two-part question leaves room, you reason, for the half the first five missed. The next places go to the next-best chunks, and those are 35 more copies of that section, scoring exactly as the first five do. Places six to ten hold copies, and the bike ticket waits behind all 40.

</details>

---

###### 4. A follow-up about the second flat

An estate agency runs an assistant on its site for people looking for a flat to rent. Your code searches the listing descriptions with each new message and pastes the top three chunks into the request, after the chat so far. A renter asks for flats near the central station with a balcony, and the reply names three. The renter writes back: "Does the second one allow cats?"

For the renter's follow-up, which chunks does the search return?

<details><summary>A. The second flat's pet rules</summary>

✗ The chat names the second flat and the model will read the chat, so you credit the search with the same view. The search reads one text, the renter's new message, and "the second one" names no flat. It returns the chunks closest to a question about cats, from any flat, and the second flat's rules make the three only if they outrank every other flat's.

</details>

<details><summary>B. Pet rules of flats near the central station only</summary>

✗ On a listings site the filters you set stay in place when you type a new search, and the chat set "near the central station" a turn ago. This search keeps nothing between turns: its query is the new message, which says nothing about the station. Flats across the city with a cat rule compete on equal terms.

</details>

<details><summary>C. Pet rules of flats from anywhere in the listings</summary>

✓ Right. The search returns the pet rules of whichever flats best match a question about cats. The second flat is among them only by chance, and nothing in the query ties any of them to it.

The model reads the chat, where the second flat is named, next to cat rules that can belong to other flats. A renter who trusts the reply turns up to the viewing with a cat carrier and hears at the door that the building allows no pets.

The query is whatever text your code hands the search, and here that is the renter's latest message alone. "The second one" points at a reply the query does not hold, so the only words left to match are about cats. Picture six listings, four of them with a cat rule: the message as typed ranks another flat's rule first, and the flat the renter meant is not among the three. Rewrite a follow-up before searching: one extra model call reads the chat and the new message and writes a question that stands alone and names the flat, and the search runs on that. Joining the last few turns to the new message is a cheaper step in the same direction.

</details>

<details><summary>D. The chunks of the three flats from the first answer</summary>

✗ Between two people, "the second one" narrows the talk to the three already named, and a follow-up feels like a question about those three. Each search here starts fresh from the text it is given and keeps no list from the turn before. The three flats of the first answer come back only if their descriptions rank highest for a message about cats.

</details>

---

###### 5. Re-indexing an updated travel policy

A company's HR bot answers from the staff handbook, and your indexing code stores each document as chunks with ids `<doc>-<n>`. HR shortens the travel policy from five chunks to three and drops first-class rail from it. You run the indexing function on the new version overnight, and it finishes without an error.

```python
def index_document(doc_id, text):
    chunks = split(text)
    store.upsert([
        {"id": f"{doc_id}-{n}", "text": chunk, "embedding": embed(chunk), "doc": doc_id}
        for n, chunk in enumerate(chunks)
    ])

index_document("travel-policy", new_policy_text)
```

What does the index hold for the travel policy the next morning?

<details><summary>A. The three new chunks</summary>

✗ Saving a file to its path replaces the whole file, and the function is called with the document's name, so a full replacement is what you expect. The store sees only the ids it is given. The new version wrote ids 0 to 2 and never named ids 3 and 4, which stay as they were, and the old rule on first-class rail is still there to be found.

</details>

<details><summary>B. The three new chunks, and the old fourth and fifth</summary>

✓ Right. The index holds five chunks for the travel policy: `travel-policy-0` to `travel-policy-2` from the new version, and `travel-policy-3` and `travel-policy-4` from the old one. The old fifth chunk still allows first-class rail on trips over four hours.

An employee asks the bot about first-class rail, and the old rule comes back with the travel policy named as its source. Finance rejects the expense claim weeks later, and the employee pays for the ticket out of pocket.

An upsert writes the records whose ids you pass and touches no other. The new version produced ids 0 to 2, which replaced the old ones in place; ids 3 and 4 were never named, so they keep their old text, embedding and `doc` field, and the count stays at 5. A filter on the document cannot tell them from the new ones. Before writing a new version, delete the document's chunks, by the `doc` field where your store deletes by a metadata filter, or keep the list of ids each document wrote and delete the ones the new version no longer produces. Either way the count goes back to 3.

</details>

<details><summary>C. The old five chunks, until the whole index is rebuilt</summary>

✗ A library index built once from a folder of vectors, then searched, teaches you that new writes wait for the next build. An upsert goes into the index the searches read, with no build step between them; a store that applies writes in the background shows them after a slight delay. By morning every search sees ids 0 to 2 from the new version beside the old 3 and 4.

</details>

---

###### 6. Two businesses on one shared index

A support platform serves several businesses from one index. Every chunk carries a `tenant` field, and each search filters on it. A bakery chain has been live for months. This morning a gym signs up, and your code imports its one-line FAQ with the same function the bakery's FAQ went through. In the afternoon a bakery customer asks when support is open.

```python
def import_faq(tenant, entries):
    store.upsert([
        {"id": f"faq-{n}", "text": entry, "embedding": embed(entry), "tenant": tenant}
        for n, entry in enumerate(entries)
    ])

import_faq("bakery", ["Support is open 8 to 4 on weekdays.", "Refunds take 5 days."])
import_faq("gym", ["Support is open around the clock."])

hits = store.search(embed("When is support open?"), k=2, filter={"tenant": "bakery"})
```

What does the bakery customer's search return?

<details><summary>A. Both of the bakery's entries</summary>

✗ The filter runs on every search, so it reads as the wall between customers. A filter only chooses among the records that exist. Both imports wrote the id `faq-0`, an id names one record in the index, and the gym's import replaced the bakery's hours. The bakery's filter then finds one record, not two.

</details>

<details><summary>B. An empty list</summary>

✗ A seed script for a test database empties its tables before it loads, and you may expect a second import to wipe the first the same way. An upsert removes nothing it is not told to touch: the gym's import wrote one record, `faq-0`, over the bakery's hours and never named `faq-1`, so the refunds entry is still in the index and the bakery's filter finds it.

</details>

<details><summary>C. The gym's round-the-clock hours under the bakery's tenant, and the bakery's refunds entry</summary>

✗ When you edit a record in most apps, the fields you leave alone stay as they were, and a write named after update invites the same picture. This upsert passes the whole record, `tenant` included, and each field it passes replaces the old one. The gym's hours now carry `"tenant": "gym"`, and the bakery's filter skips them.

</details>

<details><summary>D. The bakery's refunds entry alone</summary>

✓ Right. The search returns one chunk, `faq-1`, which reads "Refunds take 5 days." The bakery's hours are no longer in the index.

The bakery's bot stops knowing its own support hours on the day another business signs up. The bakery's support lead files a bug saying the bot forgot their hours, while the import log shows a clean run for the gym.

An id names one record in the whole index, whatever its metadata says. Both businesses built ids from position alone, `faq-0` and `faq-1`, so the gym's upsert met an existing id and replaced that record: text, embedding and `tenant` field. After both imports the index holds two records. The metadata filter did its job on what was left, and nothing of the bakery's hours was left. Put the tenant into the id, such as `bakery:faq-0`, which keeps all three records apart, or give each business its own collection, so `faq-0` in one and `faq-0` in the other are two records.

</details>

---

###### 7. Keeping HR documents away from the sales team

The intranet bot at a manufacturer answers from one index of Sales, HR and company-wide documents, each chunk tagged with a `dept` field. Among the HR chunks is this year's salary bands table, bonus caps included; assume it is the chunk nearest to any question about bonuses. `call_model` is your thin wrapper around the model call. On Monday morning an account executive asks how her bonus is calculated.

```python
SYSTEM = "The user works in Sales. Never use or reveal HR documents."

def answer(question):
    hits = store.search(embed(question), k=3)
    context = "\n\n".join(hit["text"] for hit in hits)
    return call_model(messages=[
        {"role": "system", "content": SYSTEM},
        {"role": "user", "content": f"Context:\n{context}\n\nQuestion: {question}"},
    ])
```

What does the request for her question carry?

<details><summary>A. Three chunks, all from Sales or company-wide documents</summary>

✗ The system prompt is where you put the bot's rules, and this one names the user's team and the documents to avoid. The prompt is assembled after the search has run, and the search never saw that sentence. It ranks by distance alone, so the salary bands, nearest to a question about bonuses, come back first.

</details>

<details><summary>B. Fewer than three chunks, the HR one held back</summary>

✗ On a file share, a department label comes with permissions, so you picture the store holding back the HR chunk from the three it found. The `dept` field is stored with every chunk, and this search passes no condition on it. The index returns the three nearest chunks of any department, the salary bands first.

</details>

<details><summary>C. The HR salary bands chunk among the three</summary>

✓ Right. The request carries the HR chunk with this year's salary bands, ranked first, beside two other chunks, under a system prompt that says not to use HR documents. The text she must not see is in the request the model reads.

From here, keeping the salary bands from a salesperson rests on how the model treats one sentence of its prompt. The bands also sit in every copy of that request your logging keeps, filed under her name, for anyone who reads the logs.

The search ranks chunks by distance to the query and knows nothing about who asked. The system prompt is assembled after the search has run, so it cannot change which chunks were fetched; it can only ask the model not to use one it already has. Put the permission in the search as a metadata filter, here a condition that `dept` is Sales or company-wide, and the salary bands never leave the index. Take the user's departments from your login system, not from the conversation, where anyone can type a claim.

</details>

---

###### 8. Showing sources under each answer

Staff at an accounting firm ask an assistant about tax rules, and your page turns every `[kb-...]` in the reply into a link. The model runs on the firm's own server, which logs no requests. An auditor picks an answer from last month that cites `[kb-212]` and asks you to prove that article kb-212 was in the prompt behind it. `call_model` is your thin wrapper around the model call.

```python
def answer(question):
    hits = search(question, k=3)
    context = "\n\n".join(f"[{hit.id}] {hit.text}" for hit in hits)
    reply = call_model(messages=[
        {"role": "system", "content": "Answer from the articles below. Cite the id of each article you use, like [kb-101]."},
        {"role": "user", "content": f"{context}\n\nQuestion: {question}"},
    ])
    html = re.sub(r"\[(kb-\d+)\]", r'<a href="/kb/\1">\1</a>', reply)
    save_answer(question, html)
    return html
```

What can you hand the auditor as proof?

<details><summary>A. The <code>[kb-212]</code> citation in the saved reply</summary>

✗ You told the model to cite the articles it used, and it was given three ids and no others. The citation is text the model wrote, and nothing compares it with the prompt. A reply that cites kb-145 and kb-212 after a search that returned kb-101, kb-145 and kb-180 renders both, and only one was in the prompt.

</details>

<details><summary>B. The link on the page, which opens kb-212</summary>

✗ A link that opens the right article reads like proof. What it proves is that kb-212 exists in your knowledge base. The regex turns any id the model writes into a link, so a citation to an article the search never returned renders and opens the same way.

</details>

<details><summary>C. What the model says when you ask it about that answer</summary>

✗ The assistant wrote that answer, and the author of a text is the first one you would ask about its sources. A new call carries only its own request, and last month's prompt is not in it. The model would write its list of sources the way it writes any reply, and the list could name kb-212 whether or not the search ever returned it.

</details>

<details><summary>D. Nothing in your records settles it either way</summary>

✓ Right. Nothing in your records settles it: the service saves the question and the rendered reply, and neither holds the ids the search returned. If the search returned kb-101, kb-145 and kb-180 and the reply cited kb-145 and kb-212, the page linked both all the same.

The auditor gets a link that opens a real article and no way to tell whether the assistant ever read it. When a client disputes advice built on one of these answers, the firm cannot say what the assistant read before giving it.

A citation in a plain reply is text the model wrote, chosen like every other word in it. The regex links whatever matches its pattern, and kb-212 is a real article, so the link opens. Make the citation checkable in code: save the retrieved ids with each answer, compare the cited ids with them, and drop or flag the rest; on the reply above, that leaves kb-212 cited but not retrieved. That check shows an article was in the prompt, not that it backs the sentence citing it. Some provider APIs offer citations as a feature of their own: you pass the documents as structured input, and the response points back into them. One such API parses the citations itself and documents that they always point into the documents you sent, and a provider's built-in file search returns its citations with the reply. On your own server, as here, a plain prompt gets plain text back, so the check is yours to write.

</details>

---

###### 9. Answering from a minimum wage that rose in April

Recruiters at a staffing agency ask an assistant about the current labour rules. In April the national minimum hourly wage rose from 11.40 to 12.20, and your search puts the chunk with the new figure first in every request about pay. When a request carries no retrieved text, the model gives 11.40 every time you try.

Across thousands of recruiters' questions about the minimum wage, which figure should you expect the replies to give?

<details><summary>A. 12.20 in most replies, 11.40 now and then</summary>

✓ Right. Most replies give 12.20. A few give 11.40, each with the chunk that holds 12.20 at the top of the very request it answers, and the response carries no flag on them.

A recruiter copies 11.40 an hour into a job offer for a warehouse picker, and the offer goes out below the legal minimum. The agency's payroll team finds it when the first payslip comes back for checking, weeks after the worker started.

Pasting the right chunk in makes a grounded reply more likely without making it certain. The model draws each word from the whole request and from what training left in its weights, the numbers it learned, and here training holds 11.40 from years of pages. A 2024 study, ClashEval, set six models against retrieved text that disagreed with their own answers. Where the text was right and the model's own answer wrong, each model still kept its own answer in about 2 to 4% of cases, depending on the model. The surer a model was of its answer, the less often it switched, and a model that gives 11.40 on every try without the chunk is a sure one. Those were 2024 models, so measure yours. No step between the request and the reply checks a figure against the chunk, and a claim the chunk never makes gets through the same way. Check in code: have the model quote the sentence each figure comes from, confirm the quote appears word for word in a retrieved chunk and holds the figure in the reply, and hold back a reply that fails. Checking that a cited id was retrieved shows only that the chunk was there, not that the reply follows it.

</details>

<details><summary>B. 12.20 in every reply</summary>

✗ The right figure leads every request, and putting it there was the whole point of the search. The model reads it and still draws each word from what training left in it too, where 11.40 stood for years. In a 2024 study, even the models that held on least still sided with their own wrong answer over right retrieved text about 2% of the time.

</details>

<details><summary>C. 11.40 in every reply</summary>

✗ Without the chunk the model said 11.40 every time, and that steadiness can pass for the stronger voice. Right text in the request loses seldom: in a 2024 study, six models set against right retrieved text held to their own wrong answer in about 4% of cases at most. Here the chunk is right, and every request carries it at the top.

</details>

<details><summary>D. 11.40 in the first replies, then 12.20 from there on</summary>

✗ A colleague who hears a new figure a few times stops quoting the old one, and every request here carries 12.20. Each call starts from the same weights and a request of its own, and nothing one call carries stays for the next, so the thousandth reply can give 11.40 as readily as the first.

</details>

---

###### 10. Adding a reranker to an insurer's claims assistant

An insurer gives its claims handlers an assistant that pastes the top three chunks of a vector search into each request, and the request has room for no more than three. The index holds 300,000 chunks of policy wordings, and each answer has to arrive within a few seconds. On 200 test questions whose answering clause you know, the search ranks that clause in its top three for 110, between 4th and 20th for 75, and below 20th for 15. You are adding a reranker.

Where should the reranker run?

<details><summary>A. On the three chunks the search returns</summary>

✗ The three chunks are what the model reads, so they feel like the place to spend the effort. Given three chunks, a reranker returns the same three in a new order. For the 75 questions whose clause the search ranks 4th to 20th, the clause never reaches the reranker, and their requests carry the same text as before.

</details>

<details><summary>B. On the search's top 20, keeping its best three</summary>

✓ Right. On the top 20, the reranker can bring the answering clause into the three for any of the 75 questions where the search ranks it 4th to 20th, and it still reorders the 110 it had. The 15 below 20th stay out of its reach.

With the reranker on the three alone, a handler checking a bike stolen from a locked shed keeps getting general notes on bike theft, while the clause that settles the claim sits fifth in the search, two places short of every request.

A vector search compares two embeddings made apart, one for the query and one for the chunk, so a chunk about bikes and theft in general can sit closer than the clause that answers. A reranker reads the query and the chunk together and scores how well one answers the other, and it can only reorder what it is given: on the three it returns the same three, and a clause the search ranks 31st never reaches a shortlist of the top 20. Reading a chunk with the query takes a pass of the model for each pair, which is why a reranker runs on a shortlist. BEIR, a 2021 benchmark of 18 datasets, found rerankers of this kind among the best on average on data they were not trained for, at a high cost in computation; those were 2021 models, so measure yours. Size the shortlist so the answering clause usually falls inside it, measured on questions whose answers you know, as these 200 are.

</details>

<details><summary>C. On all 300,000 chunks in the index, keeping the best three</summary>

✗ If reading 20 chunks with the query helps, reading every chunk would also catch the 15 below 20th. A reranker spends a pass of the model on each chunk it scores, so this means 300,000 passes for every question, while the vector search compares the query with embeddings computed once, when the chunks were stored. A shortlist exists to keep that reading small enough for an answer in seconds.

</details>

---

###### 11. Keyword search on an online store's help articles

An online store ranks its help articles with `build_bm25`, your wrapper that takes each article as a list of search terms and scores the articles against a query's terms with standard BM25. The articles are in English and Chinese. The Chinese one says an order cannot be cancelled once it ships, using the word 发货, "ship goods". One customer types `shipping`; another, in the Chinese app, types `发货`. The help page lists every article that scores above zero.

```python
articles = [
    "Shipping is free on orders over $50",
    "Your order shipped on Monday and arrives within three days",
    "订单发货后无法取消",
]

def search_terms(text):
    return text.lower().split()

index = build_bm25([search_terms(a) for a in articles])
for query in ["shipping", "发货"]:
    print(query, index.get_scores(search_terms(query)))
```

Which articles does the help page list for each customer?

<details><summary>A. Both English articles for <code>shipping</code>, and the Chinese one for <code>发货</code></summary>

✗ A web search box finds "shipped" when you type "shipping", and years of that teach you to expect it everywhere. That comes from a step someone set up that cuts words to a shared stem, and this index has none. Terms match only when they are equal, so `shipping` misses the Monday article, and the Chinese article, one long term, matches nothing.

</details>

<details><summary>B. Only the free-shipping article for <code>shipping</code>, and the Chinese one for <code>发货</code></summary>

✗ The Chinese article plainly contains 发货, and a lowercase split works well enough on English. `split()` cuts at whitespace, and Chinese is written without spaces, so the whole sentence becomes one term. BM25 compares whole terms, and `发货` is not that term, so all three articles score zero.

</details>

<details><summary>C. Only the free-shipping article for <code>shipping</code>, and nothing for <code>发货</code></summary>

✓ Right. `shipping` scores above zero on the free-shipping article alone, and `发货` scores zero on all three. The article saying an order shipped on Monday never reaches the English customer, and the Chinese customer finds nothing.

To English customers the search looks patchy, and to Chinese customers it looks broken. Support staff answer by hand the tickets an article already covers, one question about a parcel at a time.

Keyword search matches search terms, and the terms are whatever your `search_terms` returns. `text.lower().split()` cuts at spaces only, so `shipped` and `shipping` are two unrelated terms, and a Chinese sentence, written without spaces, comes back as one term that no query equals. BM25 then scores what it was given. Stemming, which cuts each word down to a shared root, belongs to this step: stem both sides and both words become `ship`, after which `shipping` scores above zero on both English articles. Cut Chinese into two-character pieces, or use a splitter built for it, and `发货` finds its article. Search engines split the work the same way: one common engine by default lowercases and splits words without stemming, and another stems only when you turn on its stemming step.

</details>

<details><summary>D. All three for each query, the best match first</summary>

✗ A vector search has a score for every chunk, and BM25 gives every article a score too, so you expect each to land a little above zero. BM25 adds only for the query terms an article contains, and the score is zero wherever none appears: two of the three articles for `shipping`, and all three for `发货`.

</details>

---

###### 12. Merging keyword and vector results into one list

A software company is about to give its account managers a search over the billing handbook. On every query it runs a keyword search scored by BM25 and a vector search scored by cosine similarity, and you choose how to merge the two result lists into one ranked list in which each search moves the order. For the query "cancel annual subscription refund", the three chunks that matter score as below, and no other chunk scores higher in either search.

| Chunk | Text, opening words | BM25 | Cosine |
| --- | --- | --- | --- |
| billing-03 | "Refund rules for annual plans: an annual subscription refund is prorated ..." | 5.32 | 0.756 |
| billing-07 | "To cancel a subscription open Billing and choose Cancel ..." | 3.95 | 0.998 |
| billing-12 | "You can end your plan whenever you like ..." | 0.00 | 0.955 |

Which merge do you choose?

<details><summary>A. Add the scores with equal weights, 0.5 × BM25 + 0.5 × cosine</summary>

✗ Equal weights are the natural first choice, on the idea that each search then gets an equal say. The weights multiply scores on different scales: cosine stays between -1 and 1, while BM25 here reaches 5.32. The sum gives billing-03 3.038 and billing-07 2.474, the order BM25 gives alone, and billing-12, with no keyword match, trails.

</details>

<details><summary>B. Add 1/(60 + rank) for each list a chunk appears in</summary>

✓ Right. Merged this way, billing-07 comes out on top, ahead of billing-03, with billing-12 third. Only positions count: billing-07, second by BM25 and first by cosine, beats billing-03, first by BM25 and third by cosine.

With a raw score sum, the embeddings cost money to build and query, and on queries like this one they move almost nothing. Whenever the keyword scores run large, the account managers get the merged list in keyword order, until one of them asks what the vectors are for.

Cosine similarity stays between -1 and 1. BM25 has no upper bound: it adds a term for every query word a chunk contains, so its scale moves with the query and the collection. In an equal-weight sum, billing-07 leads billing-03 by 0.242 in cosine and trails it by 1.37 in BM25, so the BM25 gap decides. Hybrid search needs a merge that puts the two lists on one footing, and rank fusion does it by position: each list adds 1/(k + rank) for a chunk, with k at 60, the value that the 2009 paper introducing it fixed in a trial run and found close to the best, though not critical. Here that gives 0.0325 for billing-07, 0.0323 for billing-03 and 0.0161 for billing-12. It is not the only fix: a 2022 analysis of fusion found that a weighted sum of scores first rescaled to a common range, its one weight tuned on a small set of examples, did better than rank fusion both in and out of the domain it was tuned on.

</details>

<details><summary>C. Keep the higher of the two scores for each chunk</summary>

✗ Taking the maximum is how you would combine two votes of confidence: whichever search is surer decides. The raw scores sit on different scales, so the bigger number wins on scale alone. Any BM25 score above 1 beats every cosine, and the result is billing-03, billing-07, billing-12, the BM25 order again.

</details>

<details><summary>D. Put the vector list first, then append the keyword hits it lacks</summary>

✗ Appending one list to the other keeps every chunk either search found, which feels like the point of running two. The order, though, is the vector list's alone: billing-07, billing-12, billing-03. BM25 decides nothing about the chunks both searches return, and billing-03, the best keyword match, comes last.

</details>

---

## Further reading

- [ClashEval](https://arxiv.org/abs/2404.10198), 2024: six models set against retrieved text that disagreed with their own answers, measured on how often they took wrong text over their own right answer, and how often the reverse.
- [BEIR](https://arxiv.org/abs/2104.08663), 2021: 18 datasets for testing retrieval on data a model was not trained for, where rerankers and late-interaction models did best on average, at a high computing cost.
- [Reciprocal rank fusion outperforms condorcet and individual rank learning methods](https://doi.org/10.1145/1571941.1572114), 2009: merging rankings by position alone gave better results than any single system it combined, and than the standard method of its day.
- [An Analysis of Fusion Functions for Hybrid Retrieval](https://arxiv.org/abs/2210.11934), 2022: rank fusion against a weighted sum of normalised keyword and vector scores, where the sum, tuned on a few examples, came out ahead.
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401), 2020: the paper that named RAG, pairing a text generator with a search over a vector index of Wikipedia.

Next: [Memory](../memory/README.md)
