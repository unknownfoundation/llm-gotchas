# Embeddings and vector search

Needs: [Tokens and tokenizers](../tokens-and-tokenizers/README.md) · Reviewed October 2026

An embedding, or vector, is a list of numbers, often a thousand or more, that an embedding model, separate from the one that writes replies, computes from a text so that texts with similar meaning get similar lists. Vector search keeps one embedding per stored text, embeds each query the same way and returns the texts whose embeddings are nearest to it.

- **Token**: a piece of text, often a word or part of one. An embedding model's input is counted in tokens, and its input limit is the most tokens it takes in for one text.
- **Query**: the text your code hands to the search, often a user's question as typed.
- **Length**: the size of a vector, the square root of the sum of its squared numbers. Picture the vector as an arrow from the point where every number is 0, with one coordinate per number: its length is how long that arrow is. It is not how many numbers the list holds.
- **Unit vector**: a vector of length 1. Normalising a vector means dividing it by its length, which gives it length 1 and keeps its direction.
- **Dot product**: multiply two vectors number by number and add up the products.
- **Cosine similarity**: the dot product divided by both lengths, which leaves only the angle between the two arrows. A search often reports it as each result's score, higher for a nearer text.
- **Distance**: how far apart two vectors are, 0 for identical ones. L2 distance, also called Euclidean distance, is the length of the straight line between the two arrow tips; cosine distance is 1 minus the cosine similarity.
- **Chunk**: one piece of a longer document, such as a section of a manual, embedded and stored as its own vector.
- **Top-k**: the k results a search returns, ranked from the one nearest the query down.
- **Metadata filter**: a condition on the metadata, the fields stored beside each vector such as the id of the customer it belongs to, that you pass to the search with the query.

Where the model runs decides who sets three things: the input limit, what happens to text over it, and whether vectors come back at length 1. On a provider's API the provider sets all three. On your own server the input limit comes with the model's files, and the server or library that runs the model decides the other two. Embedding and search work the same wherever the model runs.

A vector search can find the nearest stored vectors in two ways:

| Way | What it does |
| --- | --- |
| Exact search | compares the query with every stored vector, a full scan |
| Approximate search | compares the query with part of the store, through an index built before any query arrives: it follows links between nearby vectors, or opens only the groups of vectors nearest the query. Its search width sets how far it looks: how many candidates a link-following search keeps, or how many groups the other kind opens |

A vector is a list of numbers no one can read, so you check a search by running queries whose answers you know and seeing where those answers land. Public benchmarks rank embedding models by an average score over many tasks. One of them, the MTEB benchmark (2022), found no model best on all of its tasks, so the queries worth testing are your own users'.

This topic lets you:

- work out what one vector stands for when a page runs long or covers many topics, and which texts a customer's question will find
- change the embedding model, the way scores are computed or the lowest score a result may have in a search that already works, and know what to test again
- set up search over many customers' documents, over a store too large to scan, and for queries that hinge on a rare name or a word such as "no"

###### 1. Embedding a long policy page in one piece

An HR assistant searches the policies of a company with an embedding model you run on your own server; you have left every server setting at its default. Your code embeds each policy page whole, as one text. The travel policy has twelve sections of about equal length and runs to about 1,400 tokens, and the model's documentation gives an input limit of 512 tokens. The call returns one vector and no error.

Which part of the travel policy is that vector computed from?

<details><summary>A. The whole page</summary>

✗ The call took the whole page and returned a vector, so you assume the vector covers all of it, read at once or in pieces and blended. The model takes in at most 512 tokens, and at its default settings the server cut the rest first; reading in pieces is an opt-in setting a server may offer. Sections 6 to 12 play no part in the vector, nor does most of section 5.

</details>

<details><summary>B. The opening part only</summary>

✓ Right. The vector is computed from the first 512 tokens, about 37% of the page: sections 1 to 4 and the start of section 5. Sections 6 to 12 never reached the model, and the call did not say so.

An employee asks whether the policy pays for a partner's flight, which the last section covers. Search scores the travel page by its first four sections, none of which mention partners, so other pages can rank above it, and the assistant can answer that no policy covers partners while the employee pays for the second ticket.

An embedding model takes in at most its input limit, and a longer text cannot go in whole. A server left at its defaults that returns a vector for such a text without an error has cut it. Several servers for running embedding models cut at their defaults, keeping the start and dropping the end, and so does a common library for embedding text in your own code. What comes back is a vector of the usual size, built from the opening alone. If you change their settings, servers can also keep the end instead or refuse a long text with an error, and some can embed it in pieces and average them. On a provider's API the rule is the provider's: some require the input to stay under the limit, and others cut longer input without saying so. Wherever the model runs, count each page's tokens with the model's own tokenizer and split anything over the limit into chunks that fit, each embedded on its own.

</details>

<details><summary>C. The opening and closing parts, without the middle</summary>

✗ A long log shown on screen is often shortened by cutting out its middle, so you expect the server to keep the page's conclusion along with its start. With nothing changed, the server makes a single cut, at the input limit, and keeps only what comes before it. The conclusion sits well past that cut and goes with the middle.

</details>

<details><summary>D. The closing part only</summary>

✗ If you have trimmed a chat history to fit, you kept the newest messages, and the same habit says the end of the page survives. Servers that cut at their defaults work from the other end. They keep the text from the start up to the input limit, so the closing sections are the part that goes.

</details>

---

###### 2. Splitting the benefits page into sections

An onboarding assistant at a logistics company searches the staff handbook, and your code embeds each handbook page whole; every page fits within the embedding model's input limit. The benefits page has ten sections of similar length, from health cover to the bike scheme, and one of them is about dental care. You split that page and embed each section on its own.

For the query "does our plan pay for dental work?", how should you expect the dental section's score against the query to compare with the whole page's?

<details><summary>A. Higher than the whole page's</summary>

✓ Right. The dental section scores higher against the query than the whole page did, though the page held every word the section holds.

Before the split, a new hire asking whether a filling is covered could get a short note about the office's dentist day ranked above the benefits page, and leave without reading the dental rules at all.

A vector is one direction, and one direction cannot point at ten topics at once. The whole page's vector leans a little toward each of its ten sections and sits close to none of them; the section's vector leans toward dental care alone, as the query does. Picture the ten topics as directions at right angles to each other, as unrelated as two directions can be: a vector spread evenly over all ten scores at most 0.32 with any one of them, while a text about one topic can score near 1. In one such picture the dental query scored 0.39 against the whole page and 0.85 against the dental section. Real topics are not at right angles and a real model's numbers differ, so the picture shows the direction, not the size. Embed the handbook as chunks of one topic each, so a question about one topic meets a vector about that topic.

</details>

<details><summary>B. About the same as the whole page's</summary>

✗ The dental text is word for word the same in both, so the same match is a natural guess. A vector does not keep its sections apart. It is one direction for the whole text, pulled toward health cover, pensions and the bike scheme as much as toward teeth, while the section's vector is pulled toward teeth alone and sits much nearer the query.

</details>

<details><summary>C. Lower than the whole page's</summary>

✗ More text gives a query more to catch on, so the whole page feels like the stronger match. An embedding is not a count of matches. The page's nine other topics pull its one vector away from dental care, and the section, free of them, lies nearer the query.

</details>

---

###### 3. One search box for a manual and a forum

A home-internet provider puts its setup manual and its customer forum behind one search box, and embeds both with the model the forum already uses to flag duplicate posts. A customer types "how do I change my wifi password?". The manual's section on it lists the steps; a forum post from last year asks the same thing in other words and never got a reply.

Which of the two should you expect nearer the top?

<details><summary>A. The manual section that answers it</summary>

✗ A search by meaning should find what the customer needs, and the steps are what they need. This model was trained to put texts that say the same thing close together, which is how it flags duplicates. The steps do not say what the query says. The forum post does, in other words, and it lands nearer.

</details>

<details><summary>B. Both, at about the same score</summary>

✗ Both texts are about changing a wifi password, so a level score is a fair guess. Sharing a subject is not what this model measures; it measures whether two texts say the same thing. The post restates the query and the steps answer it, and to this model the restatement is the closer of the two.

</details>

<details><summary>C. The forum post that asks it</summary>

✓ Right. The unanswered post comes back above the manual's steps. It is the query in other words, which is what this model was trained to find.

The customer opens a question like their own with no reply under it, gives up on the search and phones the helpline, where an agent reads out the steps from the manual section that ranked lower.

An embedding places texts by whatever its model learned to treat as alike. A model trained to flag duplicate questions learned that two questions asking the same thing belong together, so a query phrased as a question finds other questions first. The passage that answers is a different kind of text, a list of steps, and it ranks close only as far as a model learned to pair questions with their answers. Finding texts like the query, such as similar questions, is one task; finding a longer passage that answers a short query is another, and each has models trained for it. For a help search, choose a model trained on questions and the passages that answer them, and keep forum posts with no reply out of the store or in a list of their own. Some models also expect a query to be embedded with a different setting from a stored text, or with a prefix, a short label such as "query:" added in front of it.

</details>

---

###### 4. Switching the wiki search to a newer embedding model

A company wiki has 20,000 pages embedded with one model. On Monday you switch to a newer model from the same maker, also 1,024 numbers per vector; its release notes say nothing about the old vectors. From then on, new and edited pages and every query go through the new model, and the old vectors stay until a page is next edited. On Thursday a support engineer searches for the on-call handbook, last edited a year ago.

Where does the year-old handbook land in the results?

<details><summary>A. Near the top, as before</summary>

✗ Same maker, same 1,024 numbers: the newer model has every mark of a drop-in upgrade. The two models learned their coordinates apart, so the new model's query and the old page sit on two unrelated grids. Their score says nothing about the match, and among 20,000 pages the handbook has no claim on a place in the top 10.

</details>

<details><summary>B. A few places lower than before</summary>

✗ You expect a newer model from the same maker to refine the old one, nudging old scores rather than replacing them. The new model's coordinates bear no relation to the old ones, so there is nothing to nudge. The score between the new query and the old page measures nothing, and the page's rank follows that number, not its text.

</details>

<details><summary>C. Nowhere; the search fails with an error</summary>

✗ You would expect the store to catch a mismatch this deep. It sees two lists of 1,024 numbers, and nothing in either names the model that made it. The search runs, scores the old page against the new query like any other pair, and returns its top 10.

</details>

<details><summary>D. At any rank from first to last</summary>

✓ Right. The year-old handbook can come back at any rank among the 20,000, and its place says nothing about how well it matches. The search runs as usual, since both vectors hold 1,024 numbers.

Pages edited since Monday rank by what they say, and every older page ranks by a number with no meaning, so the handbook can sit far below pages that have nothing to do with on-call work. The support engineer decides it was deleted and spends an afternoon writing a new one from memory.

An embedding model learns its own coordinates in training, and two models trained apart can put the same text at unrelated points, the way two maps of one city can use different grids. A score between the new model's query and an old page compares a point on one grid with a point on the other: it comes out as a number, and the number measures nothing about the match. The same maker and the same 1,024 numbers per vector do not make the grids agree. Nothing in a vector says which model made it, so the vector store has nothing to warn you with. Re-embed every page with the new model into a new store, move queries over once it is full, and keep the model's name next to each vector. Documentation for embedding models and vector stores gives the same advice: vectors from two models do not mix, unless a maker states that a series of its models shares one set of coordinates.

</details>

---

###### 5. Saving duplicate-ticket scores in a 0-to-1 column

You are adding a duplicate finder to a helpdesk, so the support team sees "possible duplicate" next to a new ticket. For each pair of tickets, your code divides both embeddings by their lengths, takes the dot product of the two results and saves it as the pair's score. The score goes into a database column declared for values from 0 to 1.

Which values can the score take?

<details><summary>A. Anything from 0 to 1</summary>

✗ A threshold such as 0.8 reads like a share of 1, and dividing by the length is where you would expect the squeeze into 0 to 1 to happen. Length 1 fixes how long each vector is, not where it points. (3, 4, 0) and (-3, -4, 0) point in opposite directions and score -1, and the insert fails.

</details>

<details><summary>B. Anything from 0 to 2</summary>

✗ 0 to 2 is the range of cosine distance, which is 1 minus the cosine and a number a store may report instead, but this code computes no distance. The score is the cosine itself, so a ticket and its exact copy score 1, never 2, and two tickets pointing opposite ways score -1.

</details>

<details><summary>C. Anything from -1 to 1</summary>

✓ Right. The score can be anything from -1 to 1: 1 for two tickets whose vectors point the same way, 0 at a right angle and -1 for opposite directions, and any pair below 0 fails the insert.

Unless the worker that saves pairs catches that failure, it stops at the first pair below 0, and tickets filed after it get no duplicate check at all. Some models put almost every pair of real texts well above 0, so tests can pass for months before such a pair arrives.

Dividing by the length makes a unit vector and leaves the direction alone, and the dot product of two unit vectors is their cosine similarity. A cosine follows the angle between the vectors, and so does its sign. Take (3, 4, 0): its length is 5, and dividing gives (0.6, 0.8, 0). Its double, (6, 8, 0), scores 1 against it; (-4, 3, 0) sits at a right angle and scores 0; (-3, -4, 0) points the other way and scores -1. Whether real ticket pairs ever fall below 0 depends on the model, and nothing in the maths stops them. Declare the column from -1 to 1 and clip each score to that range, since rounding can put a vector scored against itself a hair above 1, at 1.0000000000000002.

</details>

<details><summary>D. Any number, with no upper or lower limit</summary>

✗ That holds for a dot product of raw embeddings, which grows with both vectors' lengths, and this code does take a dot product. Dividing by the length first takes that growth out. What is left is the cosine of the angle between the two tickets, which stays between -1 and 1, apart from rounding in the last digit.

</details>

---

###### 6. Moving the FAQ search from cosine to L2 distance

You are moving a bank's FAQ search from a store that ranks by cosine similarity to one that ranks by L2 distance. Your embedding model returns every vector at length 1, and both stores run an exact search, comparing the query with every stored vector. Before the switch, a teammate asks for two days to re-check the top 10 for each of the 300 questions in your eval set.

For the same question, how does the new top 10 compare with the old one?

<details><summary>A. Identical, document for document</summary>

✓ Right. Every question gets the same ten documents in the same order. Only the scores change: the closest document now carries the smallest number instead of the largest.

The teammate can hand the two days back. What can still break the agreement is a vector stored at another length, say by an import that skipped the division, and a check of every stored vector's length takes minutes.

For two unit vectors, the squared L2 distance is 2 minus twice their cosine similarity: the two squared lengths, 1 and 1, minus twice the dot product, which for unit vectors is the cosine. Distance falls exactly as cosine rises, so sorting by smallest distance gives the order of largest cosine, close calls included. A store that ranks by dot product agrees as well, since the dot product of unit vectors is the cosine. Both stores here run exact search; an approximate index on either side could drop or swap a few results by itself, and that would come from the index, not from the measure.

</details>

<details><summary>B. A few near-tied documents swapped, the rest the same</summary>

✗ Two formulas, one a similarity and one a distance, make you brace for close calls that flip. On unit vectors the squared distance is 2 minus twice the cosine, so every close call keeps its order. Swaps come from elsewhere: a vector stored at another length, or an approximate index standing in for exact search.

</details>

<details><summary>C. Long documents lower, short ones higher</summary>

✗ L2 distance cares about length where cosine does not, and the long answers in the FAQ are the obvious ones to suffer. The length that counts is the vector's, not the text's, and every vector here has length 1 whatever the answer's size. With equal lengths, distance and cosine read the same angle, and no document moves.

</details>

<details><summary>D. Most of the ten replaced by other documents</summary>

✗ A distance and a similarity answer different questions, how far apart and how alike, so a store built on the other one ought to pick other documents. On unit vectors they are two readings of one angle. The ten closest by one are the ten closest by the other, for every question in the eval set.

</details>

---

###### 7. Ranking recipes by dot product against raw page vectors

On a recipe site, search runs an embedding model on your own server, which returns vectors without normalising them. Your code normalises each query's vector before the search. The page vectors are stored as they come, at lengths between 4 and 9, and search ranks pages by the dot product with the query and shows the first ten.

Which recipe pages does a query get in its top ten?

<details><summary>A. Cosine's top ten, in cosine's own order</summary>

✗ On unit vectors the dot product and the cosine give one order, and the rule travels with you to vectors it was never about. Here only the query is unit length. Each page's score is its cosine times its own length, and lengths from 4 to 9 move long pages ahead of closer short ones, so the first ten are not cosine's.

</details>

<details><summary>B. Near pages, with long ones pushing out closer ones</summary>

✓ Right. Pages near the query in direction still lead, but among them a long vector beats a short one that points closer, so neither the ten nor their order is cosine's.

Recipes whose vectors came out long keep crowding the first results, and a short recipe that matches better slips to the second page. The site's editor, trying a few searches, keeps meeting the same handful of recipes near the top, each one ranked above the dish that was asked for.

The dot product is the cosine similarity times both lengths. Dividing the query's vector by its length fixes one of the two, and that one is the same for every page, so it cannot change the order. Each page's length is its own, and it multiplies that page's cosine: a page at length 9 and cosine 0.6 scores 5.4, ahead of a page at length 4 and cosine 0.85, which scores 3.4. Direction still counts, since a page pointing away from the query scores 0 or below at any length. Picture 2,000 recipes of 40 kinds at lengths between 4 and 9: over 200 queries, the dot product's top 10 shared 4.6 of 10 pages with cosine's on average and never all ten, and its pages averaged length 8.0 against 6.4. Divide every page vector by its length before you store it, or rank by cosine. A provider's API that returns unit vectors is safe either way, while a library or server you run yourself may return raw vectors unless you ask it to normalise them.

</details>

<details><summary>C. The longest page vectors, whether near or not</summary>

✗ Once length enters the score, "length decides" is the quick conclusion. Direction still multiplies it: a page at length 9 that points at right angles to the query scores 0, below any short page that points toward it. The longest vectors reach the first ten only where they also point near the query.

</details>

<details><summary>D. Cosine's top ten, the same pages reordered by length</summary>

✗ With lengths between 4 and 9, you might picture length as a nudge that reshuffles pages already close and no more. Length multiplies a page's whole score, so a page outside cosine's ten can climb into the dot product's: length 9 times cosine 0.6 gives 5.4, above length 4 times cosine 0.85, which gives 3.4.

</details>

---

###### 8. Keeping the no-match cutoff after a model switch

A legal-research tool says "no good match" when even the best page has a cosine similarity below 0.5 with the question. You chose that line on the old embedding model with 200 saved questions: the library answers 150 of them and has no answer for the other 50. Now every page is re-embedded with a new model, and the release goes out on Friday.

Before Friday, which check tells you whether the 0.5 line can stay?

<details><summary>A. No check; keep 0.5 as it is</summary>

✗ A cosine always runs from -1 to 1, and on one fixed scale a line at 0.5 should mean the same wherever you draw it. Each model places its scores on that scale in its own way. The documentation of two embedding models puts their scores roughly between 0.6 or 0.7 and 1, so a line chosen on one model says nothing about another.

</details>

<details><summary>B. The 200 questions' top pages, compared with the old model's</summary>

✗ A search is judged by its rankings, and if every question gets the same pages in the same order, nothing has changed for the user. A model can keep every ranking and still move every score. If every cosine rose by the same amount, each top 10 would stay as it was, while the top page for each of the 50 questions with no answer rose past 0.5.

</details>

<details><summary>C. The 150 answered questions, each against 0.5</summary>

✗ Checking that every right page still clears 0.5 is the test you would write first, and it passes. It checks one side of the line only. If the new model scores everything higher, the 50 questions the library cannot answer clear 0.5 as well, and "no good match" never shows.

</details>

<details><summary>D. The new model's scores on both groups of questions</summary>

✓ Right. Only the new model's scores on both groups show where the line belongs: the 150 right pages on one side, the best pages for the 50 unanswered questions on the other. Kept unchecked, 0.5 can let all 50 through.

With 0.5 kept, a lawyer who asks about a statute the library does not hold gets a list of pages instead of "no good match", and may cite one of them. The page turns out to cover a neighbouring statute, and the brief goes back to be rewritten.

A cosine similarity says how alike two vectors are within one model, and each model spreads its scores in its own way. The documentation of two embedding models says their scores sit roughly between 0.6 or 0.7 and 1, so even unrelated texts can score above 0.5. A model can keep every ranking and still move every score. Picture a new model that is the old one with one extra coordinate, equal in every vector: each cosine moves halfway from its old value up to 1, and each question's top 10 stays the same. The right pages, at 0.57 to 0.84 before, now score 0.79 to 0.92, and the best pages for the unanswered questions, at 0.30 to 0.48 before, now score 0.65 to 0.74, all above 0.5. Score both groups of saved questions with the new model and set the line where they part, here between 0.74 and 0.79.

</details>

---

###### 9. Keeping one company's chunks after the search

A docs assistant serves 40 bike makers from a single vector store, and each maker's manual covers the same settings in its own words. Every chunk carries a `tenant` field in its metadata. Your code asks the store for the top five chunks, then keeps the ones from the company asking. An engineer at one of the makers asks about a setting in their own manual.

```python
def context_for(question, tenant_id):
    hits = store.search(embed(question), k=5)
    mine = [hit for hit in hits if hit["metadata"]["tenant"] == tenant_id]
    return "\n\n".join(hit["text"] for hit in mine)
```

How many of the engineer's own chunks reach the model?

<details><summary>A. None, or one at most</summary>

✓ Right. The model gets none of the engineer's chunks, or one: whichever of their chunks happened to rank in the top five across all 40 manuals. The filter removes the other makers' chunks after the search has stopped looking, and nothing refills the list.

The engineer is told the documents do not cover a setting that sits in their own manual. Their support ticket reaches the docs team, who find the setting in that maker's manual within a minute.

Top-k is a cut over the whole store, and a filter after it can only remove. With 40 manuals describing the same settings, a query about one setting lands about as close to all 40 makers' sections on it, and the five places go to five of them. Picture 2,000 chunks, 50 settings in each of 40 manuals: over 300 queries, the asking maker's section made the top five 44 times, and its median rank was 21. After the filter, 256 queries had no chunk left and 44 had one. Pass the tenant to the store as a metadata filter, so the five are chosen from that maker's chunks alone; in the same picture that returns five every time. Stores differ in where their own filter runs. Some apply it before the vector search. One documents that with approximate search it filters only after the index scan, so when a condition matches a tenth of the rows, about 4 of the 40 candidates the scan returns are left. Check in your store's documentation where its filter runs.

</details>

<details><summary>B. Five, all from their manual</summary>

✗ `k=5` is the number you asked for, and the line after the search guards against another company's chunk without, you assume, touching the count. The five are picked from all 40 manuals before that line runs. Most belong to other makers, the list comprehension throws them away, and nothing goes back for more.

</details>

<details><summary>C. Two to four of the five</summary>

✗ The engineer asked about their own manual, and their manual answers the question, so several of the five ought to be theirs. The search scores every chunk against the query alone, and the `tenant` field plays no part in that score. Thirty-nine other manuals describe the same setting, and their sections take most or all of the five places.

</details>

---

###### 10. Swapping a full scan for an approximate index

A shop's product recommendations compare the item a shopper is viewing with 100,000 item vectors, and the full scan makes the page slow. You build an approximate index. You set its search width to 10, the same as the 10 results you ask for. Before shipping, you run 500 saved queries through both.

Across the 500 saved queries, what does the index return next to the full scan's top 10?

<details><summary>A. Exactly the scan's ten, in the scan's own order</summary>

✗ A database index returns exactly what a full scan returns, only faster, and this one is called an index too. An approximate index compares the query with part of the store and skips the rest, and a true neighbour can sit in the part it skips. Those neighbours are what it gives up for speed.

</details>

<details><summary>B. The scan's ten items, in a different order</summary>

✗ If "approximate" brings rounding to mind, the right ten in a rougher order is what you expect. What the index gives up is items. A search that stops early returns the near items it reached, so some true neighbours are absent, not merely out of place.

</details>

<details><summary>C. Close items, a few not among the scan's ten</summary>

✓ Right. Close items come back, and for some queries a few of the truly closest are missing, replaced by items that are close but not the closest. A wider search brings them back, at the cost of speed.

A shopper looking at a hiking boot gets a sensible list, but the matching gaiters, among its true nearest items, never appear: the search stopped before it reached them. The loss shows only beside a full scan, query by query, and it is uneven: some queries lose nothing, while others lose several of their ten.

An approximate search finds neighbours without reading every stored vector. A graph index links each vector to a few of its neighbours, and a query walks those links, keeping a list of the best candidates found so far; the search width is the length of that list, and a short list ends the walk early, sometimes in the wrong part of the graph. A cluster index sorts the vectors into groups when it is built and opens only the groups nearest the query; there the search width is how many groups it opens, and a true neighbour in a group it skips is missed the same way. Picture 1,500 vectors in 30 groups, each vector linked to its 8 nearest neighbours and to 2 others at random: with a width of 10, the walk found 87.7% of the true top 10 over 200 queries, 89 queries got all ten, and the worst got 2, while a width of 50 found 99.7%. A wider search finds more of the true neighbours and runs slower. Measure the share of the true top 10 your index finds against exact search on your own vectors, and raise the width until that share is what you need.

</details>

<details><summary>D. Items from anywhere in the catalogue, close or not</summary>

✗ With no slack between the width and the 10 results, a search that ends anywhere in the catalogue is the natural fear. Each step of a graph walk moves toward the query, and a cluster index opens the groups nearest it, so what comes back is close to the query even where it is not the closest. A wider search narrows the gap to the scan.

</details>

---

###### 11. Searching a newspaper archive for a century-old mayor

A regional newspaper puts its archive online with two searches side by side, one by embeddings and one by keywords. A reader types the full name of a town mayor from a century ago and asks where he was born. One article names him and his birthplace; dozens of others describe other mayors' lives.

Which search should you expect to rank that article first?

<details><summary>A. The vector search</summary>

✗ Search by meaning is described as going beyond exact text matches, so it should find the right article at least as well. The meaning of this question is "where was a mayor born", and dozens of articles carry that meaning. A 2021 study found retrievers that search by embeddings far behind keyword search on simple questions about rare names.

</details>

<details><summary>B. Each search ranks it first</summary>

✗ Both searches see the same question and the same archive, so both ranking it first is a fair guess. Keyword search counts the name, found in this article alone, for far more than the common words around it. The vector search compares gists, and dozens of articles share this one. On such questions a 2021 study found keyword search well ahead.

</details>

<details><summary>C. Neither; other articles rank above it in both</summary>

✗ The question also carries "mayor", "born" and "where", words that dozens of articles hold, so you expect them to drown the name in keyword search as well. Its standard formula weighs each word by how rare it is: a name found in a single article counts for far more than words found in dozens, and it picks the article out.

</details>

<details><summary>D. The keyword search</summary>

✓ Right. The keyword search finds the article by the name itself, a word that appears in it and almost nowhere else. The vector search can rank articles about other mayors' birthplaces above it.

A reader who tries only the search by meaning scrolls through a page of other mayors' childhoods, while the one article that names this mayor and his birthplace sits below them all.

Keyword search scores an article by the words it shares with the query, and its standard formula counts a rare word for more than a common one, so a name found in one article picks that article out. An embedding compares the gist of the query with the gist of each article, and "where was this mayor born" is the gist of dozens of them. A 2021 study, Simple Entity-Centric Questions Challenge Dense Retrievers, built simple questions about people and places, such as where someone was born, and found that dense retrievers, which search by embeddings, did far worse than keyword search: they handled common names, and rare ones only when the question's wording had been seen in training. Those were 2021 models, so test yours on the names your users type. Run both and merge the two lists, which is what hybrid search does, so a rare name or a product code can still come first.

</details>

---

###### 12. Searching holiday flats for an entrance with no steps

A holiday-rental site searches its listings by embeddings. Two of its flats have descriptions that are the same except for one word: "Ground-floor flat by the harbour. The entrance has steps." and "Ground-floor flat by the harbour. The entrance has no steps." A traveller who uses a wheelchair types "a ground-floor flat with no steps at the entrance".

Which flat should you expect at the top of the results?

<details><summary>A. The flat with no steps</summary>

✗ To the traveller, "no steps" is the whole point of the query, and search by meaning is meant to catch exactly that. The two descriptions share every word but "no", so their vectors sit almost on top of each other. NevIR, a 2023 study, found most retrieval models it tested no better than chance at ranking texts that differ only by a negation.

</details>

<details><summary>B. Either of the two</summary>

✓ Right. Either flat can come first, and which one is down to the model, not to the word "no" in the query. You cannot count on the flat with no steps ranking higher.

The traveller books the first result and arrives at the harbour to find steps between the pavement and the front door, with a wheelchair and a week's luggage. The other listing, the flat they needed, said "no steps" in its second sentence.

An embedding captures what a text is about, and both listings are about a ground-floor flat by the harbour and its entrance. A negation flips the meaning while changing one short word, and that is the case NevIR, a 2023 study, built its test on: pairs of documents that differ only by a negation. It found most retrieval models, state-of-the-art ones included, ranking such pairs the same as or worse than a random ranking; models that turn the query and the document into two separate vectors came last, and models that read the query and the document as one input did best. Do not leave a must-have to the vector. Keep step-free access as metadata and apply it as a metadata filter, and if wording must decide, re-score the top results with a second model that reads the query and each listing together, and reorder them by that score. Those were 2023 models, so test yours on pairs like these.

</details>

<details><summary>C. A tie, both on the same score</summary>

✗ If embeddings ignore "no", the two listings turn into one text and should tie. The model reads "no" like any other token, so the listings get two different vectors, if only slightly, and one scores above the other. Which flat wins is set by how the model was trained, and it varies from one model to the next.

</details>

<details><summary>D. The flat with steps</summary>

✗ If you have read that embeddings miss "no", you expect the search to hear "steps at the entrance" and pick the flat that has them. The "no" sits in the query and in one listing alike, and the model reads it in both. That one word moves each vector only slightly, so a given model may put either flat first.

</details>

---

## Further reading

- [Simple Entity-Centric Questions Challenge Dense Retrievers](https://arxiv.org/abs/2109.08535), 2021: on simple questions about people and places, retrievers that search by embeddings fall far behind keyword search, and handle rare names worst.
- [NevIR](https://arxiv.org/abs/2305.07614), 2023: most retrieval models tested, state-of-the-art ones included, rank two texts that differ only by a negation no better than chance.
- [MTEB](https://arxiv.org/abs/2210.07316), 2022: embedding models compared on 58 datasets across 8 kinds of task, where no single method comes out best on all of them.
- [ANN-Benchmarks](https://arxiv.org/abs/1807.05614), 2018: approximate nearest-neighbour methods measured on how many true neighbours they find against how fast, where very different approaches reach comparable trade-offs.

Next: [Retrieval and RAG](../retrieval-and-rag/README.md)
