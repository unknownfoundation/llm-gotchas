# Prompting

Prev: [Generation parameters](../generation-parameters/README.md) · Next: [Structured outputs](../structured-outputs/README.md) · Reviewed October 2026

A prompt is everything a model reads on one call, often built by your code from templates and variables: instructions, examples and the data the task works on, in the system prompt and the messages. Before the call, the model's chat template joins these parts into one text with markers for each message, and that text is what the model reads.

- **Token**: a piece of text, often a word or part of one; prompts are counted and billed in tokens.
- **Tokenizer**: the vocabulary and rules that cut text into tokens; most model families train their own, so one text can come to different counts on different models.
- **Weights**: the numbers a model learned in training, which turn the text it reads into probabilities for the next token; an open-weight model is one whose weights are published, so you can run it on your own server.
- **System prompt**: the instructions meant to hold for the whole chat, sent as the first message of the list or, on some APIs, as a separate field.
- **Roles**: the label on each message: `system` for your instructions, `user` for the person, `assistant` for the model's own replies. Some models are trained to give the system prompt priority over the user's messages.
- **Temperature**: the model gives every possible next token a probability and one is drawn; temperature sets how strongly the draw favours the likeliest tokens, and a temperature of 0 asks for the likeliest token on every step.
- **Generation cap**: the request's limit on how many tokens the reply may run to, often called max tokens.

On a provider's API you write every part of the prompt, though the provider can add text of its own, such as instructions for calling tools; on your own server the model's chat template turns your message list into the text the model reads, and you can print that text; a ready-made app wraps what you type in a system prompt of its own.

A prompt can carry five kinds of part:

| Part | What it does |
| --- | --- |
| Instructions | say in plain words what to do, what not to do and what the reply should look like: its form, its length, the fields it holds or the answers it may pick from |
| Examples, often called few-shot examples | show sample inputs with the replies you want, placed before the real input; a prompt with none is zero-shot |
| Persona, often called role prompting | tells the model who to be, such as "You are an experienced tax adviser" |
| Step-by-step reasoning, often called chain of thought | asks the model to write out the steps of its reasoning in its reply, as well as the answer |
| Data | carries the material the task works on, such as a document or a set of records, in a layout you choose: JSON, CSV, a Markdown table or plain lines |

Whatever parts it holds, a prompt also has a **format**: the characters that set the content out without changing its words, such as the separator between a label and its text (a colon, a dash, a line break), spacing and casing. Changing the data layout goes further than changing the format. Moving records from JSON to CSV changes some of the words too, along with the characters around them.

In production one prompt runs on every input a product receives, and each call sends and bills all of it.

This topic lets you:

- keep a live product's replies true to the data and rules you give it, and decide what goes in the prompt and what your code checks
- predict what reordering a prompt or adding to it will do to the answers and to the bill, before the change ships
- work out what drives an answer, compare two instructions on a test set, and move a tuned prompt to a model's new version

###### 1. A welcome email for a booking with no check-in date

A hotel group emails each guest a week before arrival. Your code fills a prompt template from the booking record, with lines such as "Guest name:", "Check-in date:" and "Room:", and asks the model to write the email. A booking imported from a travel agent has no check-in date, so that line of the prompt has nothing after its label. The prompt says nothing about missing details.

How do you find out that this guest's email was written without a date?

<details><summary>A. An error from the call, and no email</summary>

✗ A web form refuses to submit with a required field left blank, and a check-in date is as required as fields get. The API never sees your template or its fields. It receives text, and a line with nothing after its label is valid text, so the call goes through and returns an email as usual.

</details>

<details><summary>B. A warning in the response beside the email</summary>

✗ A response already carries more than the reply, such as token counts and the reason the reply ended. Those fields describe the call, and to the API "Check-in date:" with nothing after it is just a line of text, so the email comes back with no warning beside it.

</details>

<details><summary>C. A line in every such email saying the date is missing</summary>

✗ A model can point out a missing detail, and a blank date is hard to miss. Nothing in this prompt asks it to: the reply can point out the missing date, write around it or put something plausible in its place, and your code cannot tell which without reading every email.

</details>

<details><summary>D. A check in your own code</summary>

✓ Right. Only a check your code runs, on the booking or on the filled-in prompt, can tell you. The request carries "Check-in date:" with nothing after it, the call succeeds and the response looks like every other, so the guest can get a warm welcome that names a date the booking never held.

The model reads the filled-in text and nothing else: the template, the booking record and the field names stay in your code. Here the empty value fills in as nothing, and the model reads only a label with nothing after it; where a template fills in a missing value as the word None, the model reads that word as one more value.

Check the values the email needs before you fill in the template, and hold the booking for a person when one is empty. Log the filled-in prompt next to each reply, so the text the model read can be looked up when a guest calls. On your own server you can also print the whole text the chat template builds; through a provider's API, the messages your code sends are the closest record you have.

</details>

---

###### 2. Banning rate quotes in a bank's support chat

Your code runs a bank's support chat on a model. Rates change daily, so the system prompt says: "Never quote an interest rate; send the customer to the rates page instead." Before launch you ran 300 test chats, and no reply quoted a rate. The chat now handles about 40,000 conversations a week.

What should you expect from the replies once the chat is live?

<details><summary>A. No reply quotes a rate</summary>

✗ The system prompt is where your rules live, and 300 test chats came back clean. They rule out a rule that breaks often and can easily miss one that slips once in a few hundred chats; spread over 40,000 chats a week, even a slip that rare reaches customers many times over.

</details>

<details><summary>B. Now and then a reply quotes a rate</summary>

✓ Right. Now and then a reply quotes a rate, though the system prompt forbids it, and the call returns it as a normal reply, with a rate that may already have changed.

An instruction in the system prompt is text the model reads, and on models trained to give it priority, the system role raises its weight. Each token is still drawn from probabilities: the rule makes the tokens of a rate less likely and removes none of them, and no step compares the reply with the rule. Clean tests can only show that breaks are rare. Say one chat in 200 broke the rule: a run of 300 test chats would come out clean more than one time in five, and 40,000 chats a week would carry about 200 breaks.

If a broken rule costs the business, test every reply in code before it goes out: here, a check for a percentage or a number next to the word rate, which swaps in the rates-page answer when it fires.

</details>

<details><summary>C. A reply quotes a rate only when a customer tries to trick the bot</summary>

✗ Trick prompts are the best-known way to break a bot's rules, and your 300 test customers asked plain questions. A rule can slip on a plain question too: each token is sampled from probabilities the rule lowers without bringing to zero, and a question about a mortgage is where the tokens of a rate are likely to begin.

</details>

<details><summary>D. A reply that quotes a rate comes back as an error</summary>

✗ The generation cap is a request setting the server enforces, and the rule travels in the same request. The system prompt is text the model reads, and nothing checks the reply against it, so a reply that quotes a rate comes back as an ordinary reply.

</details>

---

###### 3. Fitting a hiking app's replies on a watch

A hiking app has an assistant that answers trail questions on the phone and, from today, on a watch whose screen shows about 15 words. The phone and the watch send the same request: the system prompt "Keep replies short." with the hiker's question. On the watch, most replies run off the screen. Your code can give each device a system prompt of its own.

Which system prompt should the watch get?

<details><summary>A. "IMPORTANT: keep every reply very short."</summary>

✗ Prompts that shout their key rule in capitals are everywhere, and capitals tell a reader which rule matters most. Weight is not what this rule lacks: "very short" still has no size in it, and the request never says the screen holds 15 words.

</details>

<details><summary>B. "You are a mountain guide of few words."</summary>

✗ A persona sets the voice of every reply, and a guide of few words is the voice a watch needs. "Few words" carries no number, and the guide's few words have no reason to stop at the 15 the watch can show.

</details>

<details><summary>C. "Reply in at most 12 words."</summary>

✓ Right. The line with a number in it is the one that tells the model how long a watch reply may be, and 12 leaves room under a screen that holds about 15. The other three describe the reply, the voice or the device, and none puts a size into the text the model reads, so most watch replies still run off the screen.

The phone and the watch send the same words, so the replies come from the same probabilities on both. "Short", "very short" and "few words" carry no number, and the model's sense of them comes from its training, not from your screen; the word "watch" names a device, not a word count. A length the model can aim at is a number: a count of sentences or words.

An instruction raises the odds and guarantees nothing, so measure each reply in code before it reaches the screen, and shorten it at a sentence end when it runs over.

</details>

<details><summary>D. "The hiker is reading this on a watch."</summary>

✗ The model has read plenty about watches, small screens included. A watch is a kind of device, and screens differ: the line gives the model no count to aim at, so the reply's length still rests on what the model makes of the word.

</details>

---

###### 4. Listing the causes in an insurer's claims tool

An insurer's claims tool asks a model to pick each claim's cause from five, listed in the prompt as A to E in the order of the claim form: water damage, fire, theft, storm and accidental damage. Each month the count per cause goes to the pricing team, which sets next year's premiums from it. Before the next run, you review how the list is written.

Which way of listing the causes should the claims tool use?

<details><summary>A. The form's order on every claim</summary>

✗ The form's order has served the claims team for years, and to a person five causes are five causes in any order. The model reads the letters too, and a 2023 study found models favouring certain letters, so whichever cause sits on a favoured letter, water damage at A if A is favoured, is counted high across the claims, month after month.

</details>

<details><summary>B. The most common cause first on every claim</summary>

✗ A form for people in a hurry puts the likeliest answer first. In a 2023 study, models favoured particular letters, A and B for one, C and B for another, so moving the common cause to A pushes its count up or down by the model's lean, and the pricing team reads that lean as claims.

</details>

<details><summary>C. Temperature 0, with the form's order kept</summary>

✗ Temperature 0 is what people set for repeatable results, and steady counts are what a pricing team wants. Where a system takes the setting, it sends each pick to the likeliest letter, and the lean toward certain letters is part of what makes a letter likely, so the lean lands on whichever cause holds the favoured slot, claim after claim. Identical runs are not promised either.

</details>

<details><summary>D. A new random order on every claim</summary>

✓ Right. A new random order on every claim, with the letters mapped back to causes in your code, spreads the models' preference for certain letters over all five causes: over 1,000 claims, each cause sits at letter A on about 200 of them. With one fixed list, whichever cause holds the favoured letter is counted high month after month, and the pricing team raises that cause's premium on the strength of it.

The choices you list are part of the instructions, and the model reads their order and their letters along with their words. A 2023 study, Large Language Models Are Not Robust Multiple Choice Selectors, tested 20 models and found their answers moved when the options were reordered: the models favoured certain letters, A and B for one, C and B for another, and moving every right answer to D cut one model's accuracy by 6.3 points. Shuffling removes neither the preference nor the mistakes it causes. It spreads those mistakes over the five causes about equally, so no single cause is inflated; every count is pulled a little toward an even split instead.

Shuffle the causes on each call and map the letter back in code. Where the counts carry money, run each claim under a few orders and keep the cause that wins most often, and check your own model by running a sample of claims under two orders.

</details>

---

###### 5. Grouping a ticket sorter's examples by label

A delivery company sorts its support emails into billing, delivery and damage, and your code asks a model for each label. The prompt shows nine labelled emails before each new one, three per label. To keep it tidy, you grouped them: the three billing emails first, then the three delivery ones, then the three damage ones, right before the new email.

Once the examples are grouped, what should you expect in the labels the model writes for new emails?

<details><summary>A. More emails labelled billing than are about billing</summary>

✗ Billing opens the list, the spot that sets the tone when you skim a page yourself. A 2021 study of few-shot examples found the lean running the other way, toward answers placed near the end of the prompt. Here those are the three damage examples, right before each new email.

</details>

<details><summary>B. Each label as often as with the examples mixed</summary>

✗ Prompting guides present examples as a way to show the reply's form, and nine emails show the same form in any order. The model also reads their labels and their order, and a run of one label at the end pulls its answers toward that label. A 2021 study found that order alone could move accuracy from close to random guessing to near the top results then reported.

</details>

<details><summary>C. More emails labelled damage than are about damage</summary>

✓ Right. Expect more emails to come back labelled damage than are about damage, so some delivery emails land in the damage queue and the delivery queue looks quieter than it is. The three examples right before each new email all say damage.

The model reads the examples as part of the text before its answer, so their labels, their order and their content all pull on the label it writes next. A 2021 study, Calibrate Before Use, found that the choice of examples, and even their order alone, could move accuracy from close to random guessing to near the best results of the time, and traced this to a lean toward certain answers, among them answers that sit late in the prompt.

Mix the labels so that none closes the set, keep the set close to the mix of real mail, and compare each label's share on a sample of emails whose labels you know, before and after any change to the examples.

</details>

<details><summary>D. Every label more accurate than with mixed examples</summary>

✗ Grouped by label, the examples make a clean reference sheet, the way you would lay them out for a new colleague. The model reads them as text leading up to its answer: three damage labels right before each email push answers toward damage, and emails about billing or delivery pay for it.

</details>

---

###### 6. Example replies in a university's admissions inbox

A university's admissions office drafts replies to applicants' emails with a model. The prompt holds a one-line instruction to draft a reply, three example replies, and then the new email. All three examples were written for engineering applicants, and each ends by inviting the reader to the engineering open day on 14 March. The other faculties hold open days too.

What will replies to applicants outside engineering say about open days?

<details><summary>A. No open day in any reply</summary>

✗ You picked these examples for their tone and shape, and the open day was only what each engineering reply happened to end with. The model does not split an example into form and content: it reads all of it, and three replies ending with one invitation make that ending part of what a reply here looks like.

</details>

<details><summary>B. An invitation to the engineering open day</summary>

✓ Right. Expect some replies to history or nursing applicants to end with an invitation to the engineering open day on 14 March, an event that is not theirs. All three examples end that way, and every reply is written with them in the text before it.

The model reads every part of an example: the form, and also the content, the length and any detail the examples share. An event all three repeat is a pattern it picks up. Two makers' prompting guides warn about this: one says to vary examples enough that the model does not pick up patterns you did not intend, the other that too many examples can make replies fit them too closely.

Vary the examples in the details you do not want copied, take out what belongs to one case, such as the open day, and add it as data only to the requests it concerns.

</details>

<details><summary>C. The open day of each applicant's own faculty, with its correct date</summary>

✗ A capable new hire would spot the pattern, an invitation to an open day, and adapt it to each applicant's faculty. Adapting needs the other dates, and they appear nowhere in the request: engineering's is the only date the model reads, so an invitation to the history open day could carry only a date the request never held.

</details>

<details><summary>D. An open day only where an applicant asks about visiting</summary>

✗ A person reuses an example's detail only where it fits: the open day goes into a reply about visiting. The shared ending pulls on every reply, asked for or not: the request carries no rule that keeps the invitation for questions about visiting, and an applicant asking about fees reads the same three examples first.

</details>

---

###### 7. Adding an expert persona to a study app

A study app for nursing students answers factual questions on anatomy, drugs and hospital law. Before exam season, you add one line to the system prompt: "You are a senior hospital pharmacist with 30 years of experience." You will compare the share of right answers on your 2,000 test questions, each with one right answer, before and after.

What will the before-and-after comparison show?

<details><summary>A. About the same share as before</summary>

✓ Right. Expect the share of right answers to stay about where it was across the 2,000 questions: no clear gain on the drug questions and no clear loss on anatomy or law, while the replies may take on a pharmacist's voice.

A persona is a few tokens of text in the prompt, 13 here as counted by one widely used tokenizer. It can change the voice of the replies and adds no facts the model did not already have. A 2024 study put 162 personas into the system prompt of nine open-weight models of that time and asked 2,410 multiple-choice factual questions: on average a persona did not raise accuracy over none, the effect of any one persona was largely random, and personas matched to a question's field came out only slightly ahead of unmatched ones.

Keep the line if you like its voice, and judge accuracy by the before-and-after run, not by the persona's title.

</details>

<details><summary>B. More right answers in every subject</summary>

✗ Persona lines are standard advice in prompting guides. A persona changes the tone of the replies and adds nothing to what the model knows: a 2024 study of 162 personas on 2,410 factual questions found no gain in accuracy over no persona on average, and most personas made no significant difference either way, so expect about the same share on the 2,000 questions.

</details>

<details><summary>C. A clear gain on the drug questions</summary>

✗ Drugs are a pharmacist's trade, and on the drug questions the persona matches the work. A 2024 study compared matched and unmatched personas: the matched ones came out ahead by a margin its authors called minor, and on average no persona beat having none, so the drug questions land about where they were.

</details>

<details><summary>D. A clear loss on anatomy and law</summary>

✗ A pharmacist is out of their depth on nerves and court cases. The line adds no knowledge and removes none: in a 2024 study, personas outside a question's field trailed matched ones only slightly, and personas overall left accuracy close to where no persona had it, so anatomy and law stay close to their scores without the line.

</details>

---

###### 8. Putting the score before the reasons in an essay grader

A language school has a tool that scores each student essay from 1 to 10 and gives the reasons, and teachers pass both on to the student. The prompt asks for the score on the first line and the reasons after it, and a teacher suggests the reverse: the reasons first and the score on the last line.

In which order can the reasons the model writes change the score it gives?

<details><summary>A. In either order, the reasons can change the score</summary>

✗ A teacher grading by hand decides first and writes the number at the top, and the order she writes them in makes no difference to the grade. The model has no decision waiting in the text before its first token. With the score first, the score is written before any reason exists, and the reasons after it are written to fit it.

</details>

<details><summary>B. In neither order: the reasons only explain the score</summary>

✗ A rubric scores the essay and the comments only explain the mark, in whatever order they are written. The model writes one token at a time, each drawn from what is already written, so with the reasons first what they say sits in front of the score and can move it.

</details>

<details><summary>C. Only when the reasons come first</summary>

✓ Right. Only with the reasons first are they on the page before the score is written, so only then can they move it. With the score on the first line, the reasons are written to fit a number already there.

A reply is written one token at a time, each drawn from the prompt and everything written so far, and nothing already written changes. Step-by-step reasoning rests on this: steps written before the answer become part of what the answer comes from. With the score first, it comes from the instructions and the essay alone, and the reasons after it argue for it.

Ask for the reasons first and have your code read the score from the last line. Some models reason in text you never see before they reply, and that hidden reasoning comes first whichever order you ask for; the reasons a teacher reads still shape the score only when they come first.

</details>

---

###### 9. Reading a triage bot's reasoning to explain its choices

A city council sorts residents' repair reports, such as potholes, broken streetlights and blocked drains, with a model that sets each report's priority. Your prompt asks for step-by-step reasoning and then the priority, and the duty manager reads that reasoning to understand the choices. Each report the model sees includes the resident's district. Over a month, reports from one district came out low priority more often than the rest, yet no reasoning mentions the district.

Which check tells you whether the district drives the priority?

<details><summary>A. Reading the reasoning on that district's low-priority reports</summary>

✗ The reasoning comes first and the priority follows it, the way working is laid out in a maths answer. The priority comes from the whole input, and the district sits in that input whether or not the reasoning names it. Reasoning that never mentions the district tells you nothing either way about its pull.

</details>

<details><summary>B. Asking the model in a follow-up whether the district mattered</summary>

✗ Asking someone why they decided is how you would check a colleague's call. A follow-up call reads only the report and the earlier reply, with no record of how that reply was produced, so its answer is new text that can deny the district's part as smoothly as the reasoning left it out.

</details>

<details><summary>C. Changing the prompt so the reasoning covers every field</summary>

✗ If the district is left out, the usual fix is to make the reasoning walk through every field. Asked to cover the district, the model writes a sentence about it, more text from the same input rather than a reading of its weight, and a line saying the district made no difference can sit right above a priority it moved.

</details>

<details><summary>D. Resending the same reports with only the district swapped</summary>

✓ Right. Resending the same reports with only the district swapped, and comparing the priorities, shows whether the district moves them. Nothing else in those requests changes, so a shift in the share of low priorities larger than the spread between repeat runs has one cause.

The priority comes from everything before it: the instructions, the whole report with its district, and the reasoning. The reasoning is part of that input, not a record of how the priority was reached, so the district can move the priority without ever appearing in the text. A 2023 study, Language Models Don't Always Say What They Think, found that a hint planted in the prompt shifted the models' answers, that their step-by-step explanations systematically failed to mention it, and that when the hint pointed to a wrong answer they often argued for it.

Swap only the district on a sample of reports, run each a few times since replies vary, and compare the share of each priority.

</details>

---

###### 10. Choosing a layout for order rows in a prompt

A shop's sales assistant answers questions about last month's orders, and each request carries all 50 of them, five fields per order: order ID, date, city, total and status. Your database returns them as indented JSON. You can send them that way, as compact JSON with the spaces and line breaks taken out, as CSV, or as a Markdown table.

Which layout makes each request cheapest?

<details><summary>A. All four cost about the same</summary>

✗ Fifty orders are fifty orders, and a bill is usually for the data, not for how it is printed. The tokenizer reads the punctuation, field names and spaces as text, every character of it. Under one widely used tokenizer the same orders run from 965 tokens as CSV to 2,486 as indented JSON.

</details>

<details><summary>B. CSV costs the least</summary>

✓ Right. CSV costs the least and indented JSON the most: under one widely used tokenizer the 50 orders come to 965 tokens as CSV, 1,143 as a Markdown table, 1,537 as compact JSON and 2,486 as indented JSON, and seven chat tokenizers rank them in the same order.

Every character of the prompt is tokenized, the layout's included, so the count follows what a layout repeats. CSV names the five fields once, in a header line, and then carries only values and commas. The Markdown table also names them once and wraps each value in a vertical bar and a space: 22 tokens for the last row against 18 as CSV. Both JSON layouts repeat all five field names, with their quotes, in every one of the 50 orders, and indenting adds a line break and a run of spaces before each field. The ranking holds for flat rows like these; nested data does not fit a CSV line at all.

A cheaper layout also changes the text the model reads, so check that answers from the CSV are as good on your own questions before you switch, and count the exact text you send with the tokenizer of the model you call.

</details>

<details><summary>C. Compact JSON costs the least</summary>

✗ Minifying is the first fix for a payload that is too large, and here it takes indented JSON from 2,486 tokens to 1,537. Compact JSON still writes all five field names in every order; CSV writes them once, in its header, and comes to 965.

</details>

<details><summary>D. The Markdown table costs the least</summary>

✗ Markdown is everywhere in the prompts you have read, and a table names each field once, which puts it below both JSON layouts. It also wraps each value in a bar and a space: 1,143 tokens against CSV's 965 under one widely used tokenizer, and above CSV under all seven.

</details>

---

###### 11. Adding the customer's record to a plumber's quoting assistant

A plumbing firm has an assistant that prices each job from the firm's price list and a short job note: the parts, the hours and the call-out fee. Since the same assistant also answers customers' questions, your code now adds the customer's record to every request, after the note: past jobs with their prices and the engineer who usually visits. Some records hold one past job, others many. You rerun last month's 300 notes, whose right prices you know.

What should you expect when you rerun the 300 notes with the record added?

<details><summary>A. Fewer notes get the right price than before</summary>

✓ Right. Expect the share of right prices to fall: some notes that were priced right without the record come out wrong with it, though nothing about those jobs changed, and a quote can drift toward the price of another job on the record.

The model writes every token of the price from the whole request, needed or not. The note and the price list already hold everything the price depends on; the record adds the prices of other jobs beside them, and the request gives no sign that those numbers are beside the point. A 2023 study, Large Language Models Can Be Easily Distracted by Irrelevant Context, added one irrelevant sentence to each of a set of school maths word problems and found the two models' accuracy dramatically lower with every prompting method it tried; telling the model to ignore irrelevant information helped.

Send what the price needs and leave the rest of the record in your database, and rerun your known notes with and without it before you change the request.

</details>

<details><summary>B. Each note gets the same price as before</summary>

✗ A reader skips what does not concern them, and the engineer's name has nothing to do with the price of the job. The model has no step that sets lines aside before it writes, so the record's other prices sit beside the note while each digit of the quote is written, and a quote can drift toward one of them.

</details>

<details><summary>C. More notes get the right price than before</summary>

✗ More context is the usual advice for a better-informed answer, and everything in the record is true. The note and the price list already hold the parts, the hours and the fee, and the record adds only other jobs' prices; in a 2023 study, one irrelevant sentence added to maths problems lowered accuracy dramatically.

</details>

<details><summary>D. Wrong prices appear only for customers with a long record</summary>

✗ Models lose their place in long documents, and a long record is a long document. A 2023 study added a single irrelevant sentence to each maths problem and still found accuracy dramatically lower, so a customer with one past job on file can already get a wrong price.

</details>

---

###### 12. Comparing two instructions for a story sorter

At a news agency, a tool files each incoming news story under one of eight newsroom desks, such as sport or courts. Its prompt holds an instruction and ten example stories, all in the format the tool has always used: `Story:` and the story's text on one line, then `Desk:` and the desk's name on the next. A colleague wrote the current instruction and you have written a new one, and 400 past stories have known desks.

Which test on the 400 stories shows whether the new instruction files more of them under the right desk?

<details><summary>A. The current format, one run per instruction</summary>

✗ The current format has run for months without trouble, and both instructions run inside it. The model reads every separator in it, and a 2023 study found separators, spacing and casing alone moving accuracy by up to 76 points on one model, so a single run in one format can crown whichever instruction suits that colon.

</details>

<details><summary>B. The current format, five runs per instruction</summary>

✗ Replies vary from call to call, and averaging repeat runs is the careful habit that handles it. Repeats smooth out the noise of sampling and leave the format's pull untouched, since all five runs read the same separators. The averaged gap can still belong to the current format, and shrink or reverse in another.

</details>

<details><summary>C. Four formats, one run per instruction in each</summary>

✓ Right. Running each instruction once in each of four formats, with the words kept and only the separators, spacing and casing changed, shows whether the new instruction wins in all of them or only in the current one, where its margin may belong to the colon after `Desk`.

A colon, a dash and a line break are tokens like any word, so `Desk:` and `DESK -` give the model different inputs, though a person reads the same content in both. A 2023 study, Quantifying Language Models' Sensitivity to Spurious Features in Prompt Design, changed only such choices in few-shot prompts and found accuracy moving by up to 76 points on one open-weight model of that time; the sensitivity remained in larger models, with more examples and in models trained further to follow instructions. Its authors advise reporting a range of results across plausible formats instead of a single score.

Pick a few formats that keep every word, score both instructions in each, and switch only if the new one wins across them. Repeat runs answer a different question, how much replies vary between calls, and leave the format where it was.

</details>

<details><summary>D. Each writer's own format, one run per instruction</summary>

✗ Each writer tuned an instruction in the format they had open, and testing it as built is how you would treat any other code. The two runs then differ in the words and the separators at once, and since separators alone can move accuracy by many points, the gap cannot say which change earned it.

</details>

---

###### 13. Moving a tuned contract reader to the model's new version

At a law firm, a tool reads each incoming contract and returns its notice period and renewal date. Over three months you tuned its prompt against 500 contracts whose answers you know, and the current model gets nearly all of them right. The model's maker has released a newer version, and you plan to switch to it with the prompt unchanged.

What can you count on from the new version on your 500 contracts?

<details><summary>A. Nothing about how many it gets right</summary>

✓ Right. Nothing carries over but the text: how many of your 500 contracts the new version gets right is unknown until you rerun them, and it can come out higher on some kinds of contract and lower on others, including contracts the old version read right.

What a prompt does depends on the weights that read it. A new version is a model trained again, with weights of its own, so the same prompt gives different probabilities throughout the reply, and the wording, examples and format you tuned were fitted to the old weights. A 2023 study of prompt formats found that how well a format served one model was only a weak guide to how well it served another.

Rerun the 500 contracts on the new version before you switch, compare the answers one by one, and keep the old version until the new one matches or beats it. Through a provider's API that wait has a limit, since the provider retires old versions on its own schedule; on your own server the old weights keep running for as long as you keep them.

</details>

<details><summary>B. The same replies on all 500 contracts, word for word</summary>

✗ The prompt is the part you wrote and tested, and not a word of it has changed, so its 500 results are the prompt's own. They belong to the prompt and the model together. The new version reads the same text with weights trained again, so the old results do not travel with the prompt.

</details>

<details><summary>C. At least as many right answers as before</summary>

✗ The maker recommends the newer version and reports it scoring higher on its benchmarks. Two makers' guides for newer models warn about old prompts: one says its new model follows instructions more literally, the other that emphatic wording earlier models needed can make newer ones overreact, so a prompt tuned to the old version can lose contracts it used to get right.

</details>

<details><summary>D. The same dates and periods, in reworded replies</summary>

✗ The dates sit in the contract, and a model copying them out has nothing to get wrong. The dates are written token by token by the new weights, like the rest of the reply, and a notice period the old version read correctly can come back wrong.

</details>

---

## Further reading

- [Large Language Models Are Not Robust Multiple Choice Selectors](https://arxiv.org/abs/2309.03882), 2023: 20 models answering multiple-choice questions changed their answers when the options were reordered, leaning toward certain option letters that differed from model to model.
- [Calibrate Before Use](https://arxiv.org/abs/2102.09690), 2021: the choice of few-shot examples, and even their order alone, moved accuracy between about chance and the state of the art of the day, with a lean toward answers placed near the end of the prompt.
- [When "A Helpful Assistant" Is Not Really Helpful](https://arxiv.org/abs/2311.10054), 2024: 162 personas in the system prompt, tested on 2,410 factual questions across four model families, did not improve accuracy over no persona on average.
- [Language Models Don't Always Say What They Think](https://arxiv.org/abs/2305.04388), 2023: step-by-step explanations that leave out the cue that swayed an answer, and argue for wrong answers the cue pushed the model toward.
- [Large Language Models Can Be Easily Distracted by Irrelevant Context](https://arxiv.org/abs/2302.00093), 2023: one irrelevant sentence added to school maths word problems lowered accuracy dramatically, and an instruction to ignore irrelevant information helped.
- [Quantifying Language Models' Sensitivity to Spurious Features in Prompt Design](https://arxiv.org/abs/2310.11324), 2023: changing only separators, spacing and casing in few-shot prompts moved accuracy by up to 76 points on one model, and how well a format suited one model was a weak guide to another.

Prev: [Generation parameters](../generation-parameters/README.md) · Next: [Structured outputs](../structured-outputs/README.md)
