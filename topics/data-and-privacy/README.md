# Data and privacy

Needs: [Context window](../context-window/README.md) · [Embeddings and vector search](../embeddings-and-vector-search/README.md) · [Retrieval and RAG](../retrieval-and-rag/README.md) · [Security](../security/README.md) · [Testing and evals](../testing-and-evals/README.md) · [Where models run](../where-models-run/README.md) · Reviewed October 2026

Personal data enters an LLM feature as text: what a user types and the records and documents your system adds to a request. Privacy then comes down to where each copy of that text sits, who can read it and what deletes it.

- **Personal data**: anything that can be tied to a person, such as a name, an email, an address or a health detail, or a mix of details that together point to one person.
- **Stored reply**: some provider APIs keep each reply on their server, so a follow-up can send only the new message and the stored reply's id, which is called chaining.
- **Trace**: the record of every call made for one user request or one agent run: what each call sent, what came back and how long it took.
- **Embedding**: a list of numbers an embedding model computes from a text, so that texts with similar meaning get similar lists.
- **Index**: the store a search runs over, holding each chunk of a document with its text, its embedding and metadata.
- **Fine-tuning**: training a model further on your own examples, which changes its weights (the numbers it computes its replies with) so that its replies follow them.
- **Scrubbing**, often called redaction: replacing the personal data found in a text with placeholders before the text is stored or sent.
- **Pseudonymising**: replacing a name, an email or another identifier with a stand-in, such as a code or a hash, that is the same every time that identifier appears.

Where a request travels, and who keeps a copy of it:

- **A provider's API**: every request leaves your network for the provider's machines, and what the provider keeps, and for how long, depends on its terms, your account and the features you use.
- **Your own server**: requests to the model stay on hardware you run, and you decide what your server, your code and your libraries keep.
- **A ready-made app**: a chat app or a coding assistant decides what goes into each request, sends it to wherever its model runs, and keeps its own history of your sessions.

The path is the same on all three: each request goes to wherever the model runs.

A copy of a request's text can stay behind in these places:

| Place | What it holds | Where you find it |
| --- | --- | --- |
| Provider logs | the requests and replies a provider records after a call, for as long as its terms say | a provider's API, and the provider behind a ready-made app |
| Stored replies and files | replies, uploaded files and other data a feature keeps on the provider's side on purpose | some provider APIs |
| Your logs and traces | what your code, your libraries and your server write down about each call | a provider's API or your own server |
| Caches | answers or results kept so a repeated question skips the model call | a provider's API or your own server |
| Indexes | chunks of your documents with their text and embeddings, kept for search | a provider's API or your own server; the built-in file search of some provider APIs |
| Fine-tuned models | the weights of a model trained further on your examples | some provider APIs; your own server |
| App histories | the chats and sessions an app saves, on your device or on its maker's servers | a ready-made app |
| Exports and backups | copies of any of the above, made for another team, another company or a restore | a provider's API or your own server |

Beside a provider's API or your own server, your code writes these copies the same way. Any outside service that a tool calls keeps copies of its own. A ready-made app keeps its copies in its own history.

This topic lets you:

- follow a customer's message, a file or a secret to every place it lands when a coding assistant, an in-house model or a provider's storage handles it
- judge what a debug log, a scrubbed transcript or a hashed export tells someone about who the customers are
- carry out an erasure request, or size up a breach, across caches, traces, indexes and a fine-tuned model

###### 1. Letting a coding assistant explore a payments service

You ask a coding assistant in your terminal to find out why a test fails in a payments service. It runs on a model behind a provider's API, and you have set no rule about which files it may open. Its log shows that, while looking around, it opened `.env`, the file that holds the production database password.

What does the provider receive from that file?

<details><summary>A. Nothing; the file stays on your laptop</summary>

✗ The assistant runs in your terminal, saves its sessions on your disk and shows its work on your screen, so it passes for a local tool. The model it talks to is not local. Every step is a request to the provider, and the file the assistant opened goes out in the next one.

</details>

<details><summary>B. The file's name and nothing else</summary>

✗ A search through a project usually starts with a list of file names, and a name is all a listing needs. Opening a file is different: to use what is in it, the model has to read it, and the model reads only what a request carries, so the contents go out.

</details>

<details><summary>C. The password, encrypted and unreadable to the provider</summary>

✗ The connection to the provider is encrypted, and the padlock reads as a promise that no one else can see the data. That encryption protects the request on its way. At the provider it is decrypted, since the model works on plain text and cannot read anything still encrypted.

</details>

<details><summary>D. The password, as plain text it can read</summary>

✓ Right. The provider receives the whole file, password included, as plain text inside the assistant's next request.

The password now sits on machines your company does not run, in the provider's copy of that request for as long as its terms keep one, and in the session transcript on your laptop. The security team's only safe move is to rotate it today.

Everything the model reads arrives in a request. A file the assistant opens becomes a tool result in its message list, and the whole list goes to wherever the model runs, here the provider's machines. The connection is encrypted on the way and decrypted on arrival, since the model reads plain text. What the provider keeps afterwards sits in its provider logs, for anything from no time at all to a month or more, depending on its terms and your plan. App histories hold it as well: the assistant keeps the session on your disk, file contents and all. Keep secrets out of the folders an assistant can open, deny it such files where its settings allow, and treat any secret it has read as exposed. With the model on your own server, the file would have stayed on hardware you run.

</details>

---

###### 2. Moving a clinic's chat model onto your own server

A clinic's patient chat answers from the clinic's own notes. To keep patient messages inside the building, you move the chat model onto your own server. To find the right notes, the search still uses a provider's embedding API. The index it searches runs on your own hardware.

What leaves the building each time a patient sends a message?

<details><summary>A. Nothing; the whole chat stays inside</summary>

✗ The move was made for exactly this, and the model that reads the patient's message and writes the answer now sits in your rack. The search is a second model call with its own route. To find notes for a message it has to embed the message, and the embedding API is outside.

</details>

<details><summary>B. The message, as text the provider reads</summary>

✓ Right. The patient's message goes to the embedding provider as text, once per message, so it can be turned into numbers for the search.

A patient who types her symptoms and her date of birth has sent them to an outside company, though the model that answers her never leaves your rack. The review that approved the move listed the chat model and stopped there.

An embedding model computes an embedding from a text where the model runs, so the text has to reach it first. Moving the chat model in-house keeps its requests at home, and every other model call keeps its own route. Here each message goes out once to be embedded, and any copy the provider keeps sits in its provider logs. The notes went out the same way when the index was built, since a search compares embeddings made by one model. The answer and the notes found for it stay inside, as only the chat model reads them. Run the embedding model on your own server too, or count the embedding API among the places patient data goes.

</details>

<details><summary>C. The message's embedding, never its words</summary>

✗ What comes back from the call is a list of numbers, and that list is all your index stores, so numbers can pass for all that crosses. The numbers are computed on the provider's side, from the message, and the provider can compute them only after it has received the message's text.

</details>

<details><summary>D. The message, the notes found and the reply</summary>

✗ A service that sees part of a chat can be pictured seeing all of it, the way a chat provider would. The embedding call carries one thing, the text to embed. The notes are found by your own index, the reply is written by your own server, and neither is sent out.

</details>

---

###### 3. Erasing a user's chats on a fitness coaching app

A fitness coaching app runs on a provider's API that stores each reply on its server for 30 days. Each follow-up sends only the new message and the id of the stored reply before it. A user who told the coach about her injuries and her weight over the past two weeks asks for her data to be erased, and your code deletes her chats from your database that day.

The day after that deletion, what of her chats is still on the provider's side?

<details><summary>A. Her messages and the coach's replies, every turn</summary>

✓ Right. Every turn of her chats is still on the provider's server, her messages and the coach's replies, stored under the reply ids your code chained to.

Her erasure request is closed as done. Her injuries and her weight stay with the provider, where anyone holding your project's key can fetch them by id until the stored replies expire.

A follow-up that sends only a new message and an id still gets an answer that knows what she said before. The model reads only its request, so the provider has to rebuild that request from what it stored, her messages included. Under chaining, the provider's store holds the whole conversation, her messages as well as the coach's replies. Deleting your copy leaves that store alone. Stored replies expire after a period the provider sets, from a day to several weeks depending on the provider and the plan, and each can be deleted by its id. Have your erasure code delete every stored reply in a user's chains, or send the history yourself and turn storing off.

</details>

<details><summary>B. The coach's replies, none of her messages</summary>

✗ The feature is described as storing replies, so you picture a shelf of the model's answers with your users' words kept at home. Yet the last follow-up sent only her new message and an id, and the coach answered knowing what she had said earlier. Her earlier messages came from the provider's store.

</details>

<details><summary>C. The coach's last reply in each chat, nothing earlier</summary>

✗ Each call names only the latest id, as if every new reply took the place of the one before. Each stored reply points back to the one it chained to, and every one of them is kept until it expires or your code deletes it, so the whole chain stays.

</details>

<details><summary>D. Nothing; each chat went when she closed it</summary>

✗ A web session's data usually goes when the user logs out, and a chat she has closed is over as far as she can see. Stored replies are not tied to a session. They stay for a period the provider sets, from a day to several weeks, or until your code deletes them by id.

</details>

---

###### 4. Uploading discharge letters under a zero-retention arrangement

A hospital's account with its model provider has a zero-retention arrangement, signed off after a security review. A new feature uploads each discharge letter to the provider's file storage, so the provider's built-in file search can answer a nurse's questions about it. Your code sets no expiry on the files.

What stays with the provider from each letter?

<details><summary>A. Nothing, once each answer is back</summary>

✗ The arrangement was the point of the security review, and its name promises that nothing stays. It covers the prompts and replies that pass through a call. A file you ask the provider to store is a separate feature, kept on purpose so that later searches can read it.

</details>

<details><summary>B. The letter, for thirty days</summary>

✗ Thirty days is the figure some providers give for keeping request logs, so it is the number that comes to mind for anything sent. That clock belongs to request logs, and a letter kept for search has no such clock: its text stays until your code deletes it or an expiry you set runs out.

</details>

<details><summary>C. The letter, until you delete it</summary>

✓ Right. The text of each letter stays with the provider until your code deletes it.

A year later the provider still holds every discharge letter the wards have uploaded. When a patient asks which companies hold her records, the hospital's honest answer now includes its model provider.

Zero retention covers what passes through a call: the provider keeps no copy of your prompts and replies once the call returns. A stored file does not pass through. Keeping it is the feature, since every later search needs it, and two providers' terms leave file storage outside their zero-retention arrangements and keep a file until you delete it or it reaches an expiry you set. A third provider drops the uploaded file after two days and keeps what its search built from it, the same text in pieces, until you delete that. Stored replies and files are the copies a provider keeps on purpose, so check which of them your arrangement covers before you build on one, and delete each letter once its questions are answered, or set an expiry when you upload it.

</details>

<details><summary>D. The letter, until the search has indexed it</summary>

✗ A search you build yourself often keeps only chunks and their embeddings once a document is indexed, and a provider's file search could work the same way. What its search builds from the letter holds the letter's text, and that stays until you delete it, whatever happens to the upload itself.

</details>

---

###### 5. Logging every request while debugging a support bot

A shop that sells furniture runs a support bot that answers from resolved tickets, and every ticket still carries the name, address and phone number of the customer who raised it. `call_model` is your thin wrapper around the model call: it takes the messages and returns the reply text. While you chase a bad answer, each request goes into the shared logs that the whole engineering team can search. A customer wants an order sent to a new address and asks the bot how.

```python
def answer(customer_id, question):
    tickets = ticket_index.search(question, k=3)
    notes = "\n\n".join(ticket.text for ticket in tickets)
    messages = [
        {"role": "system", "content": "Answer using these resolved tickets:\n\n" + notes},
        {"role": "user", "content": question},
    ]
    log.info("customer=%s request=%s", customer_id, json.dumps(messages))
    return call_model(messages=messages)
```

What does the log line for this one question contain?

<details><summary>A. The question and the asker's id, nothing more</summary>

✗ A request log reads as a record of what the user sent, and the user sent one short question. What the line writes, `messages`, is everything your code put into the request, and its system prompt holds the full text of all three tickets.

</details>

<details><summary>B. The asker's own past tickets, no one else's</summary>

✗ Account pages show each customer their own history, and a search tied to the customer would be the natural design. This search gets the question and nothing about who asked. It scores the whole archive against the question alone, so the three tickets it returns can come from anyone.

</details>

<details><summary>C. Three customers' names, addresses and phones</summary>

✓ Right. The line holds the full text of three tickets, each with the name, address and phone number of the customer behind it, beside the asker's id and question.

Every engineer who searches the logs can now read the home addresses of customers who have not spoken to the bot in months. The log platform keeps those lines for its whole retention period, long after the bad answer is fixed.

A logged request is the whole request, and when a bot answers from retrieved text, most of that request is other people's text: here the system prompt holds all three tickets. The search picks tickets by how close they are to the question, never by who wrote them, so over a week the log gathers personal data from everyone whose ticket was retrieved. That makes your logs and traces a store of personal data like any other, with its own retention and its own readers. Log ids and lengths, or the ids of the tickets retrieved, and keep full requests only where few people can read them and they are deleted within days.

</details>

<details><summary>D. The ids of three tickets, without their text</summary>

✗ Logging an id rather than the record is a good habit, and it is the fix this bot needs. The code does not do it: `notes` joins the text of each ticket into the system prompt, and `json.dumps(messages)` writes that text into the log word for word.

</details>

---

###### 6. Scrubbing a pharmacy chat before it is logged

A pharmacy's chat passes every message through a scrubber before the message is logged or sent to the model. The scrubber is a set of regular expressions, text patterns written for email addresses, phone numbers and card numbers. Your tests show each of them replaced with a placeholder. A customer writes her full name, her street address and her phone number, and asks whether her insulin can arrive before Friday.

What of her personal data reaches the log?

<details><summary>A. Her name, address and medicine, with the phone number replaced</summary>

✓ Right. The log gets her name, her street address and her insulin as she typed them; only her phone number becomes a placeholder.

A pharmacist checking the logs for a failed order reads who she is, where she lives and what she takes, on a line the scrubber has already passed. The model received the same text, name and street included.

A regular expression matches text by its shape: characters around an @, a run of digits of a certain length. Her phone number has such a shape, and the scrubber has a pattern for it. Names and medicines have no shape at all, and no pattern here was written for streets, so scrubbing by pattern catches the identifiers that look like formats and lets the rest through, the mix of details that points to one person included. A scrubbed log is still personal data. Log only what you need, keep logs that hold messages under short retention and narrow access, and test the scrubber on real messages, not only on the formats it was written for.

</details>

<details><summary>B. None; placeholders stand in for every detail</summary>

✗ The tool is called a scrubber and runs on every message, and every test you wrote came back with placeholders. The tests checked the formats the patterns were written for. No pattern here was written for a name, a street or a medicine, and all three pass through.

</details>

<details><summary>C. Her medicine, with name and address replaced</summary>

✗ Name and address sit beside email and phone in every contact form, so you expect a scrubber to treat them as one group. The patterns match shapes, not meanings: the phone number matches one, while her name and her street match none and reach the log with the medicine.

</details>

<details><summary>D. Her name and address, with the medicine replaced</summary>

✗ Health details are what a privacy review asks about first, and a scrubber built for privacy ought to catch them. It knows no categories, only patterns, and a word such as insulin fits none of them. The medicine reaches the log with her name and address.

</details>

---

###### 7. Handing hashed chat logs to an analytics firm

An online shop sends a month of support chats to an outside analytics firm, and your code first replaces each customer's email with its SHA-256 hash. The same firm already sends the shop's newsletter, so it holds the mailing list of customer emails.

What can the firm tell about who wrote the chats?

<details><summary>A. Nothing about who wrote any chat</summary>

✗ Hashing is the step you reach for when an identifier has to go, and a column of hex strings shows the firm nothing it can read. No one can run a hash backwards; anyone can run it forwards, and the firm hashes each email on its mailing list and finds the same values in the chats.

</details>

<details><summary>B. Which chats have the same writer, not who that is</summary>

✗ That is the job you meant the hash to do: count repeat customers without their names. It does that, and more for anyone holding emails. A firm that holds the emails can hash them too, and each value it finds in the export names the writer of those chats.

</details>

<details><summary>C. Every writer's email, whether on its list or not</summary>

✗ A value made from an email can seem like something a clever enough tool could turn back into it. A hash cannot be reversed. The firm learns the writers it can match against emails it holds or can guess, and a writer it has never heard of stays behind the hash.

</details>

<details><summary>D. Who wrote each chat, for writers on its list</summary>

✓ Right. The firm can name the writer of every chat whose email is on its mailing list: it hashes each address on the list and looks for the same value in the export.

A customer who complained twice about a late refund keeps getting the newsletter from a firm that now knows the complaints were hers, though the shop described the export as anonymous.

A hash has no key, and the same email gives the same hash every time. That is what makes it useful for joining records, and it is also why it hides nothing from anyone who already holds the email. Replacing an identifier with its hash is pseudonymising, and pseudonymised data stays personal data for whoever can make the match; an address not on the list can often be guessed from a name and a few common formats. Exports and backups are copies like any other, and an export carries every identifier you left in it. Give each customer a random code and keep the table that maps codes to emails inside, use a hash keyed with a secret that never leaves, or leave the identifier out when the analysis does not need it.

</details>

---

###### 8. Deleting an uploaded lease from a document assistant

A document assistant lets tenants upload a lease and ask questions about it, and your own server runs the model. Your code keeps four stores: `files` for the uploaded text, `index` for its chunks, `trace` for a record of each model call, and `answers` for cached replies. A tenant uploads her lease, asks about her rent and her deposit, then removes the lease with the app's delete button, which calls `delete_document`.

```python
def upload(doc_id, text):
    files.put(doc_id, text)
    index.add(doc_id, split(text))

def ask(doc_id, question):
    if (doc_id, question) in answers:
        return answers[(doc_id, question)]
    chunks = index.search(question, doc=doc_id, k=3)
    messages = [{"role": "user", "content": "\n".join(chunks) + "\n\n" + question}]
    reply = call_model(messages=messages)
    trace.write(messages=messages, reply=reply)
    answers[(doc_id, question)] = reply
    return reply

def delete_document(doc_id):
    files.delete(doc_id)
    index.delete(doc=doc_id)
```

Once the lease is deleted, which places still hold its terms?

<details><summary>A. None; the delete cleared every copy</summary>

✗ `delete_document` clears the stored file and all of its chunks in the index, the two places the code put the lease, and the model never leaves your own server. Asking about the lease made more copies: `trace.write` stored the chunks with each request, and `answers` stored each reply.

</details>

<details><summary>B. The trace and the answer cache</summary>

✓ Right. The trace holds the chunks of her lease that went into each call, and the cache holds the replies, which state her rent and her deposit.

Weeks later an engineer searching the trace for a slow call scrolls past the lines of her lease that set her rent and her deposit.

A delete reaches the stores your code names, and this one names two. Each question made copies in two more: the trace keeps every request with the chunks pasted into it, and the cache keeps every reply, which repeats her terms in the model's words. A reply written from a document is as much personal data as the document. The model itself keeps nothing between calls; caches and your logs and traces do the keeping. Keep a list of every store a document's text can reach, key each copy by the document, and delete them together, backups included, or keep traces for days rather than months.

</details>

<details><summary>C. The trace, the cache and the model's own memory</summary>

✗ A person who has read a lease remembers it, and a model that has read one could be pictured the same way. A model keeps nothing between calls: each call reads its request and is done, and on your own server nothing writes the lease into the model. The copies are in the trace and the cache.

</details>

<details><summary>D. Only the trace; the cached replies hold none</summary>

✗ The cache holds the model's replies, and a reply is the model's own words rather than her document. Those words are her terms: one reply states her rent, the other her deposit. The cache holds them under her document's id, and `delete_document` never touches it.

</details>

---

###### 9. Keeping only the vectors in a hosted index

A clinic's search keeps the text of each chunk of its patient notes in its own database. Only the embeddings go to an index that a provider hosts. Each chunk is a sentence or two, and the embedding model is an open one that anyone can download. One morning the index's provider reports a breach.

What can whoever took the embeddings recover from them?

<details><summary>A. Nothing they can use, only rows of numbers</summary>

✗ An embedding is a row of a thousand or so decimals, and nothing in it resembles a letter. Those numbers were computed from the text, and they keep enough of it to work backwards from. A 2023 study rebuilt clinical notes from their embeddings closely enough to recover 89% of the full names in them.

</details>

<details><summary>B. Which notes say similar things, none of their words</summary>

✗ Comparing closeness is what embeddings are for, so it is natural to take that as all they give away. A model can be trained to invert an embedding model that anyone can run, and in a 2023 study such a model turned embeddings back into the texts behind them.

</details>

<details><summary>C. Every note, word for word</summary>

✗ If numbers can bring back words, it is a short step to expect every word back. On clinical notes, the 2023 study rebuilt 26% word for word and recovered 89% of the full names. Whoever holds the embeddings gets much of the notes back, not a clean copy of the database.

</details>

<details><summary>D. Much of the notes' wording, names included</summary>

✓ Right. Whoever holds the embeddings can rebuild much of each chunk's wording, patients' names among it.

The breach notice the clinic sends its patients has to cover their notes too: anyone who downloads the same open-weight model can train a second model to turn the embeddings back into text, and read sentences about who was treated for what.

An embedding is computed from the text so that texts of similar meaning get similar lists, and to do that it has to keep a good deal of what the text says. A 2023 study, Text Embeddings Reveal (Almost) As Much As Text, trained a model that guesses a text, embeds the guess and corrects it step by step; on 32-token inputs it rebuilt 92% exactly, and on clinical notes it rebuilt 26% word for word and recovered 89% of the full names. Short chunks and an embedding model anyone can run are this clinic's conditions too. Those were 2023 embedding models, so measure yours. Protect indexes like the text they came from: the same access rules, the same breach plan and the same deletion.

</details>

---

###### 10. Erasing a customer from a fine-tuned support model

A phone company fine-tuned a model on two years of support chats, with customers' names, addresses and account numbers still in the text, so its replies sound like its best agents. A customer asks to be erased. You delete her chats from the database and from every training file, and the fine-tuned model keeps answering customers.

What is left of her chats?

<details><summary>A. Pieces of her chats, in replies to any customer</summary>

✓ Right. Her chats shaped the model's weights during training, and deleting the files afterwards changed nothing in them. Pieces of what she wrote, her address or her account number among them, can come out in a reply to any customer.

Months after her erasure request was closed, another customer asking about a billing error can get a reply that carries her account number, and there is no file left to delete that would stop it.

Fine-tuning changes the weights to fit the examples, and the weights are kept apart from the examples, so deleting the examples leaves the weights as they were. Weights can hold pieces of training text word for word. In a 2020 study, Extracting Training Data from Large Language Models, researchers pulled hundreds of verbatim sequences out of a trained model by querying it, names, phone numbers and email addresses among them, some of which appeared in only one document of its training data. A 2023 study of models fine-tuned on case law, health records and email found that scrubbing the training text first reduced such leaks without stopping them. Those findings come from models tested in 2020 and 2023, so measure yours. Fine-tuned models carry what they were trained on, and the sure way to take her out is to train the model again without her chats; keep personal details out of training data from the start.

</details>

<details><summary>B. Nothing, once the files are deleted</summary>

✗ Deleting a customer's records is how an erasure request is closed everywhere else in your systems, and the training files were the records here. Training changed the weights to fit her chats, and deleting a file afterwards does not undo a training run.

</details>

<details><summary>C. A style learned from the chats, none of her words</summary>

✗ Fine-tuning is usually described as teaching a model a style, so you picture it keeping a manner of speaking and dropping the details. Trained models also keep pieces of their training text word for word: a 2020 study pulled names, phone numbers and email addresses out of one, some seen in only one document.

</details>

<details><summary>D. Her details, in replies to her own account only</summary>

✗ Chat apps that remember facts keep them per account, and a model tuned on her chats could be expected to know her the same way. A fine-tuned model has no accounts: every request reads the same weights, so what her chats left in them can come out in a reply to anyone.

</details>

---

## Further reading

- [Text Embeddings Reveal (Almost) As Much As Text](https://arxiv.org/abs/2310.06816), 2023: a model that corrects its guess step by step rebuilds most short texts word for word from their embeddings.
- [Extracting Training Data from Large Language Models](https://arxiv.org/abs/2012.07805), 2020: hundreds of verbatim sequences, personal details among them, pulled out of a trained model by querying it.
- [Analyzing Leakage of Personally Identifiable Information in Language Models](https://arxiv.org/abs/2302.00539), 2023: scrubbing the training text reduces but does not stop personal data leaking from a fine-tuned model.

Next: [Fine-tuning](../fine-tuning/README.md)
