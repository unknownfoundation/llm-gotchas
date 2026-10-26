# Prompting

Needs: [Tokens and tokenizers](../tokens-and-tokenizers/README.md) · [Generation parameters](../generation-parameters/README.md) · Reviewed October 2026

A prompt is everything a model reads on one call, often built by your code from templates and variables: instructions, examples and the data the task works on, in the system prompt and the messages. Before the call, the model's chat template joins these parts into one text with markers for each message, and that text is what the model reads.

- **Token**: a piece of text, often a word or part of one; prompts are counted and billed in tokens.
- **Tokenizer**: the vocabulary and rules that cut text into tokens; most model families train their own, so one text can come to different counts on different models.
- **Weights**: the numbers a model learned in training, which turn the text it reads into probabilities for the next token; an open-weight model is one whose weights are published, so you can run it on your own server.
- **System prompt**: the instructions meant to hold for the whole chat, sent as the first message of the list or, on some APIs, as a separate field.
- **Roles**: the label on each message: `system` for your instructions, `user` for the person, `assistant` for the model's own replies. Some models are trained to give the system prompt priority over the user's messages.
- **Temperature**: the model gives every possible next token a probability and one is drawn; temperature sets how strongly the draw favours the likeliest tokens, and a temperature of 0 asks for the likeliest token on every step.
- **Generation cap**: the request's limit on how many tokens the reply may run to, often called max tokens.

Who writes which part of the text the model reads:

- **A provider's API**: you write every part of the prompt, though the provider can add text of its own, such as instructions for calling tools.
- **Your own server**: the model's chat template turns your message list into the text the model reads, and you can print that text and read it.
- **A ready-made app**: the app wraps what you type in a system prompt of its own, and you write only your part.

How a model reads a prompt is the same on all three.

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

✗ A response already carries more than the reply, such as token counts and the reason the reply ended, so a note about a blank field would sit well beside them. Those fields describe the call, not your data. To the API, "Check-in date:" with nothing after it is a line of text, and nothing in it says a value was meant to follow.

</details>

<details><summary>C. A line in every such email saying the date is missing</summary>

✗ A careful assistant would ask about a gap like that, and a model can mention one too. Nothing in this prompt asks it to, and the reply is sampled token by token from what the prompt makes likely: it can point out the missing date, write around it or put something plausible in its place. Your code cannot tell which happened without reading every email.

</details>

<details><summary>D. A check in your own code</summary>

✓ Right. Only a check your code runs, on the booking or on the filled-in prompt, can tell you. The request carries "Check-in date:" with nothing after it, the call succeeds, and the response has the same fields as every other one.

A guest flying in for a wedding can get a warm welcome that names a check-in night nobody booked, and arrive a day early at a front desk whose system expects them tomorrow.

The model reads the filled-in text and nothing else: the template, the booking record and the names of its fields stay in your code. Here the empty value fills in as nothing, so the prompt holds the label "Check-in date:" and then the next line, and no part of the request marks a value as missing; where a template fills in a missing value as the word None, the model gets that word where a date belongs and reads it as one more value. Nothing between your code and the model knows which values the template expected, so a reply written from a prompt with a gap looks the same on arrival as every other reply. Check the values the email needs before you fill in the template, and hold the booking for a person when one is empty. Log the filled-in prompt next to each reply, so the text the model read can be looked up when a guest calls. On your own server you can also print the whole text the chat template builds, gap included; through a provider's API, the messages your code sends are the closest record you have.

</details>

---

###### 2. Banning rate quotes in a bank's support chat

Your code runs a bank's support chat on a model. Rates change daily, so the system prompt says: "Never quote an interest rate; send the customer to the rates page instead." Before launch you ran 300 test chats, and no reply quoted a rate. The chat now handles about 40,000 conversations a week.

What should you expect from the replies once the chat is live?

<details><summary>A. No reply quotes a rate</summary>

✗ The system prompt is where your rules live, and 300 clean test chats feel like proof. They rule out a rule that breaks often, and say little about one that slips once in a few hundred chats, which a test run that size can miss entirely. Spread over 40,000 chats a week, even a slip that rare reaches customers many times over.

</details>

<details><summary>B. Now and then a reply quotes a rate</summary>

✓ Right. Now and then a reply quotes a rate, though the system prompt forbids it, and the call returns it as a normal reply. Nothing in the response marks it as breaking the rule.

Weeks later a customer forwards a screenshot of a fixed rate the chat quoted, a rate that had already changed when the reply was written, and asks the bank to honour it.

An instruction in the system prompt is text the model reads, and its role gives it weight, not force. Each token of the reply is still drawn from probabilities: the rule makes the tokens of a rate less likely and removes none of them, and nothing in the request or the response compares the reply with the rule. Clean tests show breaks are rare, not that they cannot happen. Say one chat in 200 broke the rule. A run of 300 test chats would then come out clean more than one time in five, and 40,000 chats a week would carry about 200 breaks. If a broken rule costs the business, test every reply in code before it goes out: here, a check for a percentage or a number next to the word rate, which swaps in the rates-page answer when it fires. On a provider's API and on your own server alike, the system role, or a model trained to rank your instructions first, gives the rule more weight and still checks nothing.

</details>

<details><summary>C. A reply quotes a rate only when a customer tries to trick the bot</summary>

✗ Trick prompts are the best-known way to break a bot's rules, and your 300 test customers asked plain questions. A rule can slip on a plain question too: each token is sampled from probabilities the rule lowers without bringing to zero, and a plain question about a mortgage is where the tokens of a rate are likely to begin with.

</details>

<details><summary>D. A reply that quotes a rate comes back as an error</summary>

✗ The generation cap is a request setting the server does enforce, so you may file a rule sent with the request beside it. The system prompt is not a setting. It is text the model reads, and no part of the request or the response compares the reply with it, so a reply that quotes a rate comes back as an ordinary reply.

</details>

---

###### 3. Fitting a hiking app's replies on a watch

A hiking app has an assistant that answers trail questions on the phone and, from today, on a watch whose screen shows about 15 words. The phone and the watch send the same request: the system prompt "Keep replies short." with the hiker's question. On the watch, most replies run off the screen. Your code can give each device a system prompt of its own.

Which system prompt should the watch get?

<details><summary>A. "IMPORTANT: keep every reply very short."</summary>

✗ You have seen prompts that shout their key rule in capitals, and a rule marked important seems bound to weigh more. Weight is not what this rule lacks. "Very short" is still a feeling with no size in it, and the request never says the screen holds 15 words, so how short is short enough is left to the model's own sense of the phrase.

</details>

<details><summary>B. "You are a mountain guide of few words."</summary>

✗ A persona sets the voice of every reply, and a guide of few words is the voice a watch needs. A voice is not a length. "Few words" carries no number, the request still says nothing about the screen, and the guide's few words have no reason to stop at the 15 the watch can show.

</details>

<details><summary>C. "Reply in at most 12 words."</summary>

✓ Right. The line with a number in it is the one that tells the model how long a watch reply may be, and 12 leaves room under a screen that holds about 15. The other three describe the reply, the voice or the device, and none of them puts a size into the text the model reads.

On a ridge in thickening cloud, a hiker raises a wrist to check the way down and gets the first half of a sentence about two paths, with the turn they need cut off the screen.

Nothing in the prompt describes the screen: the phone and the watch send the same words, so the replies come from the same probabilities on both. "Short", "very short" and "few words" carry no number, and the model's sense of them comes from its training, not from your screen; the word "watch" names a device, not how many words its screen holds. Instructions say what the reply should look like, its length included, and a length the model can aim at is a number: a count of sentences or words. An instruction raises the odds and guarantees nothing, so measure each reply in code before it reaches the screen, and shorten it at a sentence end when it runs over.

</details>

<details><summary>D. "The hiker is reading this on a watch."</summary>

✗ The model has read plenty about watches, so naming the device seems to tell it everything about the screen. A watch is a kind of device, not a size: screens differ, and the line gives the model no count to aim at. With no number in the request, how short a reply comes out rests on what the model makes of the word.

</details>

---

###### 4. Listing the causes in an insurer's claims tool

An insurer's claims tool asks a model to pick each claim's cause from five, listed in the prompt as A to E in the order of the claim form: water damage, fire, theft, storm and accidental damage. Each month the count per cause goes to the pricing team, which sets next year's premiums from it. Before the next run, you review how the list is written.

Which way of listing the causes should the claims tool use?

<details><summary>A. The form's order on every claim</summary>

✗ The form's order has served the claims team for years, and to a person reading the list, five causes are five causes in any order. The model reads the letters too, and a 2023 study found models favouring certain option letters. A fixed order hands that preference to the same cause on every claim, month after month, and the counts carry it.

</details>

<details><summary>B. The most common cause first on every claim</summary>

✗ A form for people in a hurry puts the likeliest answer first, and the same courtesy seems to suit a model. The model does not skim a list the way a hurried person does. In a 2023 study, models favoured particular letters, A and B for one, C and B for another, so the first slot can add to the common cause or take from it, depending on the model.

</details>

<details><summary>C. Temperature 0, with the form's order kept</summary>

✗ Temperature 0 is what people set when they want repeatable results, and steady counts are what a pricing team asks for. Where a system takes the setting, each pick goes to the likeliest letter, and the models' preference for certain letters is part of what makes a letter likely. That preference stays in every pick, aimed at whichever cause holds the favoured slot, and identical runs are not promised either.

</details>

<details><summary>D. A new random order on every claim</summary>

✓ Right. A new random order on every claim, with the letters mapped back to causes in your code, spreads the models' preference for certain letters over all five causes, so no single cause collects the extra picks every month. Over 1,000 claims, each cause sits at letter A on about 200 of them.

With one fixed list, whichever cause holds the favoured letter is counted high month after month, and the pricing team raises next year's premium for that cause on the strength of it.

The choices you list are part of the instructions, and the model reads their order and their letters along with the words. A 2023 study, Large Language Models Are Not Robust Multiple Choice Selectors, tested 20 models on multiple-choice questions and found their answers moved when the options were reordered. The models favoured certain option letters, and which ones differed by model: one favoured A and B, another C and B. Moving every right answer to D cut one model's accuracy by 6.3 points. The authors traced this to the letters themselves: the models gave some letters extra weight before weighing what the options said. A fixed order, the form's or one by frequency, points that preference at the same cause on every claim. Temperature 0 only sends each pick to the likeliest letter, and the favoured letter is part of what makes a letter likely; it does not promise the same letter on every run either. Shuffling removes neither the preference nor the mistakes it causes: a claim it pushes onto the favoured letter still lands on a wrong cause. Those mistakes now land on the five causes about equally, which pulls every count a little toward an even split instead of piling them onto one cause. Shuffle the causes on each call and map the letter back in code; where the counts carry money, run each claim under a few orders and keep the cause that wins most often. Those were 2023 models, so check yours: run a sample of claims under two orders and compare the counts.

</details>

---

###### 5. Grouping a ticket sorter's examples by label

A delivery company sorts its support emails into billing, delivery and damage, and your code asks a model for each label. The prompt shows nine labelled emails before each new one, three per label. To keep it tidy, you grouped them: the three billing emails first, then the three delivery ones, then the three damage ones, right before the new email.

Once the examples are grouped, what should you expect in the labels the model writes for new emails?

<details><summary>A. More emails labelled billing than are about billing</summary>

✗ Billing opens the list, the spot that sets the tone when you skim a page yourself. A 2021 study of few-shot examples found the lean running the other way, toward answers placed near the end of the prompt. Here those are the three damage examples, right before each new email.

</details>

<details><summary>B. Each label as often as with the examples mixed</summary>

✗ Prompting guides present examples as a way to show the reply's format, and nine emails show the same format in any order. The model also reads their labels and the order they come in. A 2021 study found that the order of the examples alone could move accuracy from close to random guessing to near the top results then reported.

</details>

<details><summary>C. More emails labelled damage than are about damage</summary>

✓ Right. Expect more emails to come back labelled damage than are about damage. The three examples right before each new email all say damage, and a 2021 study found that the models it tested leaned toward answers placed near the end of the prompt.

Parcels that went missing without a scratch land in the damage queue, the claims team spends its mornings sending them back, and the delivery queue looks quieter than it is.

The examples teach more than the reply's shape. The model reads them as part of the text before its answer, so their labels, their order and their content all pull on the label it writes next. The 2021 study, Calibrate Before Use, found that the choice of examples, and even their order alone, could move accuracy from close to random guessing to near the best results of the time, and traced this to a lean toward certain answers, among them answers that sit late in the prompt. Here the end is a block of damage labels. Mix the labels so that none closes the set, keep the set close to the mix of real mail, and compare each label's share on a sample of emails whose labels you know, before and after any change to the examples. That study is from 2021, so the comparison on your own mail is what speaks for the model you run.

</details>

<details><summary>D. Every label more accurate than with mixed examples</summary>

✗ Grouped by label, the examples make a clean reference sheet, the way you would lay them out for a new colleague. The model reads them as text leading up to its answer, not as a sheet to look things up in. Three damage labels right before each email push answers toward damage, and emails about billing or delivery pay for it.

</details>

---

###### 6. Example replies in a university's admissions inbox

A university's admissions office drafts replies to applicants' emails with a model. The prompt holds a one-line instruction to draft a reply, three example replies, and then the new email. All three examples were written for engineering applicants, and each ends by inviting the reader to the engineering open day on 14 March. The other faculties hold open days too.

What will replies to applicants outside engineering say about open days?

<details><summary>A. No open day in any reply</summary>

✗ You picked these examples for their tone and shape, and the open day was only what each engineering reply happened to end with. The model does not split an example into form and content: it reads all of it, and three replies ending with one invitation make that ending part of what a reply here looks like.

</details>

<details><summary>B. An invitation to the engineering open day</summary>

✓ Right. Expect some replies to history or nursing applicants to end with an invitation to the engineering open day on 14 March. All three examples end that way, and every reply is written with them in the text before it.

A nursing applicant takes a train to the campus on 14 March, finds a hall full of robotics stands, and writes to the office asking where the nursing talks are.

Examples show sample inputs with the replies you want, and the model reads all of them: the form, and also the content, the length and any detail they share. An event that all three examples repeat is a pattern the model picks up, and every request carries it in the text the reply is written from. Two makers' prompting guides warn about this: one says to vary examples enough that the model does not pick up patterns you did not intend, and the other that too many examples can make the model fit its replies too closely to them. Vary the examples in the details you do not want copied, take out what belongs to one case, such as the open day, and add it as data only to the requests it concerns.

</details>

<details><summary>C. The open day of each applicant's own faculty, with its correct date</summary>

✗ A capable new hire would spot the pattern, an invitation to an open day, and adapt it to each applicant. Adapting needs the other dates, and they appear nowhere in the request: the model reads the prompt and the email, and engineering's is the only date in either. An invitation to the history open day could carry only a date the request never held.

</details>

<details><summary>D. An open day only where an applicant asks about visiting</summary>

✗ A person reuses an example's detail only where it fits, and expects the model to sort the examples the same way. The shared ending pulls on every reply, asked for or not: nothing in the request ties the invitation to questions about visiting, and an applicant asking about fees reads the same three examples first.

</details>

---

###### 7. Adding an expert persona to a study app

A study app for nursing students answers factual questions on anatomy, drugs and hospital law. Before exam season, you add one line to the system prompt: "You are a senior hospital pharmacist with 30 years of experience." You will compare the share of right answers on your 2,000 test questions, each with one right answer, before and after.

What will the before-and-after comparison show?

<details><summary>A. About the same share as before</summary>

✓ Right. Expect the share of right answers to stay about where it was across the 2,000 questions: no clear gain on the drug questions, and no clear loss on anatomy or law.

Students hear a pharmacist's assured voice on every answer, the wrong ones about nerves and court rulings included, and revise from them in the last week before the exam.

A persona is a few tokens of text in the prompt, 13 here as counted by one widely used tokenizer. It can change the voice of the replies, and it adds no facts the model did not already have. A 2024 study put 162 different personas into the system prompt of nine open-weight models from four families and asked 2,410 multiple-choice factual questions: on average, adding a persona did not raise accuracy over no persona, and the effect of any one persona was largely random. Personas matched to a question's field came out only slightly ahead of unmatched ones, and even the best-performing personas were no better than none. The nine were open-weight models available in 2024, so keep the line if you like its voice and let your own eval set settle any claim about accuracy.

</details>

<details><summary>B. More right answers in every subject</summary>

✗ Persona lines are standard advice in prompting guides, and they do change the tone of the replies, though tone adds nothing to what the model knows. A 2024 study of 162 personas on 2,410 factual questions found no gain in accuracy over no persona on average, and most personas made no significant difference either way.

</details>

<details><summary>C. A clear gain on the drug questions</summary>

✗ A pharmacist should know drugs best, so a gain where the persona's field meets the question is the least you would bet on. A 2024 study did compare personas matched to a question's field with unmatched ones: the matched ones came out ahead by a margin its authors called minor, and on average no persona did better than having none.

</details>

<details><summary>D. A clear loss on anatomy and law</summary>

✗ A pharmacist is out of their depth on nerves and court cases, and a model told to be one might carry that narrowness into every answer. The line adds no knowledge and removes none. In a 2024 study, personas outside a question's field trailed matched ones only slightly, and personas overall left accuracy close to where no persona had it.

</details>

---

###### 8. Putting the score before the reasons in an essay grader

A language school has a tool that scores each student essay from 1 to 10 and gives the reasons, and teachers pass both on to the student. The prompt asks for the score on the first line and the reasons after it, and a teacher suggests the reverse: the reasons first and the score on the last line.

In which order can the reasons the model writes change the score it gives?

<details><summary>A. In either order, the reasons can change the score</summary>

✗ A teacher grading by hand decides first and writes the number at the top, so the order on the page seems cosmetic. The model has no decision waiting before its first token. With the score first, the score is written before any reason exists, and the reasons after it are written to fit it.

</details>

<details><summary>B. In neither order: the reasons only explain the score</summary>

✗ A rubric scores the essay and the comments only explain the mark, so the reasons can seem like decoration in either order. The model writes its reply one token at a time, and text already written is part of what each next token comes from. With the reasons first, what they say sits in front of the score as it is written, and it can move the score.

</details>

<details><summary>C. Only when the reasons come first</summary>

✓ Right. Only with the reasons first are they on the page before the score is written, so only then can they move it. With the score on the first line, the reasons that follow are written to fit a number already there.

A student who got a 4 reads three tidy paragraphs on why, and the teacher who checks them finds every point fair, though none of them existed when the 4 was written.

A reply is written one token at a time, each drawn from the prompt and everything written so far, and nothing already written changes. Step-by-step reasoning rests on this: steps written before the answer become part of what the answer comes from. With the score first, it comes from the prompt and the essay alone, and the reasons after it are written with that score in front of them, so they argue for it. The other order turns this round: the score comes after the reasons, with them in front of it, so what they noticed can move it. Ask for the reasons first and have your code read the score from the last line. Some models are built to reason in text you never see before they reply, and that hidden reasoning comes first whichever order you ask for; the reasons a teacher reads still shape the score only when they come first.

</details>

---

###### 9. Reading a triage bot's reasoning to explain its choices

A city council sorts residents' repair reports, such as potholes, broken streetlights and blocked drains, with a model that sets each report's priority. Your prompt asks for step-by-step reasoning and then the priority, and the duty manager reads that reasoning to understand the choices. Each report the model sees includes the resident's district. Over a month, reports from one district came out low priority more often than the rest, yet no reasoning mentions the district.

Which check tells you whether the district drives the priority?

<details><summary>A. Reading the reasoning on that district's low-priority reports</summary>

✗ The reasoning comes first and the priority follows it, so it reads like the bot's working, laid out for review. The priority comes from the whole input, and the district sits in that input whether or not the reasoning names it. Reasoning that never mentions the district tells you nothing either way about its pull.

</details>

<details><summary>B. Asking the model in a follow-up whether the district mattered</summary>

✗ Asking someone why they decided is how you would check a colleague's call. A follow-up call reads only the text of the report and the earlier reply, with no record of how that reply was produced. Its answer is new text about the report, produced the same way as the reasoning, and it can deny the district's part as smoothly as the reasoning left it out.

</details>

<details><summary>C. Changing the prompt so the reasoning covers every field</summary>

✗ If the district is left out, reasoning that walks through every field ought to catch it. Asked to cover the district, the model writes a sentence about it, and that sentence is more text written from the same input, not a reading of how much the district weighed. A line saying the district made no difference can sit right above a priority it moved.

</details>

<details><summary>D. Resending the same reports with only the district swapped</summary>

✓ Right. Resending the same reports with only the district swapped, and comparing the priorities, shows whether the district moves them. Nothing else in those requests changes, so a shift in the share of low priorities larger than the spread between repeat runs has one cause.

The duty manager signs off on the triage after a month of sensible reasoning about deep potholes and dark junctions. In one district, a broken streetlight waits weeks longer for a crew, and no line of that reasoning ever says why.

The priority is written after the reasoning, and it comes from everything before it: the prompt, the whole report with its district, and the reasoning. The reasoning is part of that input, not a record of how the priority was reached, so the district can move the priority without ever appearing in the text. A 2023 study, Language Models Don't Always Say What They Think, found that step-by-step explanations can misrepresent the true reason for an answer: a hint planted in the prompt shifted the models' answers, the explanations systematically failed to mention it, and when the hint pointed to a wrong answer the models often wrote reasoning that argued for it. Those were 2023 models, and the swap test measures yours: swap only the district on a sample of reports, run each a few times since replies vary, and compare the share of each priority. Your code builds the requests, so the same test runs against a provider's model or one on your own server.

</details>

---

###### 10. Choosing a layout for order rows in a prompt

A shop's sales assistant answers questions about last month's orders, and each request carries all 50 of them, five fields per order: order ID, date, city, total and status. Your database returns them as indented JSON. You can send them that way, as compact JSON with the spaces and line breaks taken out, as CSV, or as a Markdown table.

Which layout makes each request cheapest?

<details><summary>A. All four cost about the same</summary>

✗ Fifty orders are fifty orders, and it feels as if you pay for the data, not the way it is printed. The tokenizer reads the punctuation, field names and spaces as text, every character of it. Under one widely used tokenizer the same orders run from 965 tokens as CSV to 2,486 as indented JSON.

</details>

<details><summary>B. CSV costs the least</summary>

✓ Right. CSV costs the least of the four, and indented JSON the most. Under one widely used tokenizer the 50 orders come to 965 tokens as CSV, 1,143 as a Markdown table, 1,537 as compact JSON and 2,486 as indented JSON, and seven chat tokenizers rank them in the same order.

Left as the database returns it, every question a sales manager asks pays for 2.0 to 2.6 times the tokens of the CSV, and those extra tokens count against the request's size limit as well as the bill.

Every character of the prompt is tokenized, the layout's included, so the count follows what a layout repeats. CSV names the five fields once, in a header line, and then carries only values and commas. The Markdown table also names them once, and adds a vertical bar and a space around each value: 22 tokens for a row against 18 as CSV. Both JSON layouts repeat all five field names, with their quotes, in every one of the 50 orders, and indenting adds a line break and a run of spaces before each field. Under the seven tokenizers, compact JSON runs 1.4 to 1.6 times the CSV count and indented JSON 2.0 to 2.6 times. The ranking holds for flat rows like these; nested data does not fit a CSV line at all. A cheaper layout is half the choice: it also changes the format the model reads, so check that answers from the CSV are as good on your own questions before you switch, and count the exact text you send with the tokenizer of the model you call. On a provider's API the gap is billed; on your own server it costs memory and time instead.

</details>

<details><summary>C. Compact JSON costs the least</summary>

✗ Minifying is the first fix for a payload that is too large, and it does help: indented JSON at 2,486 tokens drops to 1,537. Compact JSON still writes all five field names in every order. CSV writes them once, in its header, and comes to 965.

</details>

<details><summary>D. The Markdown table costs the least</summary>

✗ Markdown is everywhere in the prompts you have read, so a table in it is a natural pick for a lean prompt. The table does name each field once, which keeps it far below either JSON layout, but each value comes wrapped in a vertical bar and a space. Its 1,143 tokens sit above CSV's 965 under one widely used tokenizer, and above CSV under all seven.

</details>

---

###### 11. Adding the customer's record to a plumber's quoting assistant

A plumbing firm has an assistant that prices each job from the firm's price list and a short job note: the parts, the hours and the call-out fee. Since the same assistant also answers customers' questions, your code now adds the customer's record to every request, after the note: past jobs with their prices and the engineer who usually visits. Some records hold one past job, others many. You rerun last month's 300 notes, whose right prices you know.

What should you expect when you rerun the 300 notes with the record added?

<details><summary>A. Fewer notes get the right price than before</summary>

✓ Right. Expect the share of right prices to fall: some notes that were priced right without the record come out wrong with it, though nothing about those jobs changed.

A customer who called about a dripping tap gets a quote close to last winter's boiler repair, and the office manager who signs off quotes goes back to checking every one by hand.

Data carries the material the task works on, and the model writes every token of the price from all of it, needed or not. The note and the price list already hold everything the price depends on. The record adds the prices of other jobs next to them, and nothing in the request marks those numbers as beside the point. A 2023 study, Large Language Models Can Be Easily Distracted by Irrelevant Context, added one irrelevant sentence to each of a set of school maths word problems and found the models' accuracy dramatically lower, with every prompting method it tried; telling the model to ignore irrelevant information improved the results. A single sentence that did not change the answer was enough to do it. That was early 2023 and two models, so your own rerun of known notes, with and without the record, is the measure for the model you call. Send what the price needs and leave the rest of the record in your database. On a provider's API the record is also billed on every call.

</details>

<details><summary>B. Each note gets the same price as before</summary>

✗ A reader skips what does not concern them, and the engineer's name has nothing to do with a dripping tap. The model has no step that sets lines aside before it writes: every line of the request feeds each token of the reply. A 2023 study found one irrelevant sentence in a maths word problem enough to lower accuracy dramatically.

</details>

<details><summary>C. More notes get the right price than before</summary>

✗ More context is the usual advice for a better-informed answer, and everything in the record is true. True is not the same as needed: the note and the price list already hold the parts, the hours and the fee, and the record adds only the prices of other jobs. In a 2023 study, adding one irrelevant sentence to maths problems lowered accuracy dramatically for both models tested, whatever the prompting method.

</details>

<details><summary>D. Wrong prices appear only for customers with a long record</summary>

✗ A long document is where you expect a model to lose its place, so a two-line record seems safe and a two-page one risky. A short record carries the same kind of risk. The 2023 study added a single irrelevant sentence to each maths problem and still found accuracy dramatically lower, so a customer with only one past job on file can already get a wrong price.

</details>

---

###### 12. Comparing two instructions for a story sorter

At a news agency, a tool files each incoming news story under one of eight newsroom desks, such as sport or courts. Its prompt holds an instruction and ten example stories, all in the format the tool has always used: `Story:` and the story's text on one line, then `Desk:` and the desk's name on the next. A colleague wrote the current instruction and you have written a new one, and 400 past stories have known desks.

Which test on the 400 stories shows whether the new instruction files more of them under the right desk?

<details><summary>A. The current format, one run per instruction</summary>

✗ The current format has run for months without trouble, so you treat it as a neutral stage that serves both instructions alike. The model reads every separator in it, though, and a 2023 study of few-shot prompts found separators, spacing and casing alone moving accuracy by up to 76 points on one model. A single run in one format can crown whichever instruction happens to suit that colon.

</details>

<details><summary>B. The current format, five runs per instruction</summary>

✗ Replies vary from call to call, and averaging repeat runs is the careful habit that handles it. Repeats smooth out the noise of sampling and leave the format's pull untouched, since all five runs read the same separators. The averaged gap can still belong to the current format, and shrink or reverse in another.

</details>

<details><summary>C. Four formats, one run per instruction in each</summary>

✓ Right. Running each instruction once in each of four formats, with the words kept and only the separators, spacing and casing changed, shows whether the new instruction wins in all of them or only in the current one.

The margin you ship on may belong to the colon after `Desk`. When someone later tidies the template, that lead can vanish, and the night editor finds court reports landing in the sports desk's queue.

Format changes the tokens while leaving the words alone: a colon, a dash and a line break are tokens like any word, so `Desk:` and `DESK -` give the model different inputs, though a person reads the same content in both. A 2023 study, Quantifying Language Models' Sensitivity to Spurious Features in Prompt Design, changed only such choices in few-shot prompts and found accuracy moving by up to 76 points on one open-weight model; the sensitivity remained in larger models, with more examples and in models trained further to follow instructions. Its authors advise reporting a range of results across plausible formats instead of a single score. A gap between two instructions measured in one format can therefore belong to that format. Pick a few formats that keep every word, score both instructions in each, and switch only if the new one wins across them. Those were open-weight models of 2023, so the range on your own stories is what decides. Repeat runs answer a different question, how much replies vary between calls, and they leave the format where it was.

</details>

<details><summary>D. Each writer's own format, one run per instruction</summary>

✗ Each writer tuned an instruction in the format they had open, and testing it as built is how you would treat any other code. The two runs then differ in the words and in the separators at once, and the 2023 study of prompt formatting found separators alone moving accuracy by many points. The gap cannot say which of the two changes earned it.

</details>

---

###### 13. Moving a tuned contract reader to the model's new version

At a law firm, a tool reads each incoming contract and returns its notice period and renewal date. Over three months you tuned its prompt against 500 contracts whose answers you know, and the current model gets nearly all of them right. The model's maker has released a newer version, and you plan to switch to it with the prompt unchanged.

What can you count on from the new version on your 500 contracts?

<details><summary>A. Nothing about how many it gets right</summary>

✓ Right. Nothing carries over but the text: how many of your 500 contracts the new version gets right is unknown until you rerun them, and it can come out higher on some kinds of contract and lower on others.

A firm that switches on release day can send a client a renewal reminder after the notice window has closed, on a contract the old version had right.

A prompt is text, and what it does depends on the weights that read it. A new version is a model trained again, with weights of its own, so the same prompt gives different probabilities throughout the reply: the wording, the examples and the format you tuned were fitted to the old weights. The score on your 500 contracts measured the old version reading this prompt, and the new version needs a measurement of its own. A 2023 study of prompt formats found that how well a format served one model was only a weak guide to how well it served another. Rerun the 500 contracts on the new version before you switch, compare the answers one by one, and keep the old version until the new one matches or beats it. Through a provider's API that wait has a limit: the provider retires old versions on its own schedule, and requests to a retired version stop working. On your own server the old weights keep running for as long as you keep them.

</details>

<details><summary>B. The same replies on all 500 contracts, word for word</summary>

✗ The prompt is the part you wrote and tested, and not a word of it has changed, so the results feel like a property of the prompt. They belong to the prompt and the model together. The new version reads the same text with weights trained again, and nothing in your prompt carries the old results across.

</details>

<details><summary>C. At least as many right answers as before</summary>

✗ The maker recommends the newer version, and a recommended upgrade that does worse on your contracts is not what anyone plans for. Two makers' guides for their newer models say otherwise about old prompts: one warns that its new model follows instructions more literally, so prompts written for earlier models may not work at once, and the other that emphatic wording earlier models needed can make its newer ones overreact.

</details>

<details><summary>D. The same dates and periods, in reworded replies</summary>

✗ The dates sit in the contract, and finding them feels like reading rather than writing, so an upgrade seems able to change only how the reply is worded. The dates are written token by token like the rest of the reply, by the new weights. A notice period the old version read correctly can come back wrong when the new one weighs a clause differently.

</details>

---

## Further reading

- [Large Language Models Are Not Robust Multiple Choice Selectors](https://arxiv.org/abs/2309.03882), 2023: 20 models answering multiple-choice questions changed their answers when the options were reordered, leaning toward certain option letters that differed from model to model.
- [Calibrate Before Use](https://arxiv.org/abs/2102.09690), 2021: the choice of few-shot examples, and even their order alone, moved accuracy between about chance and the state of the art of the day, with a lean toward answers placed near the end of the prompt.
- [When "A Helpful Assistant" Is Not Really Helpful](https://arxiv.org/abs/2311.10054), 2024: 162 personas in the system prompt, tested on 2,410 factual questions across four model families, did not improve accuracy over no persona on average.
- [Language Models Don't Always Say What They Think](https://arxiv.org/abs/2305.04388), 2023: step-by-step explanations that leave out the cue that swayed an answer, and argue for wrong answers the cue pushed the model toward.
- [Large Language Models Can Be Easily Distracted by Irrelevant Context](https://arxiv.org/abs/2302.00093), 2023: one irrelevant sentence added to school maths word problems lowered accuracy dramatically, and an instruction to ignore irrelevant information helped.
- [Quantifying Language Models' Sensitivity to Spurious Features in Prompt Design](https://arxiv.org/abs/2310.11324), 2023: changing only separators, spacing and casing in few-shot prompts moved accuracy by up to 76 points on one model, and how well a format suited one model was a weak guide to another.

Next: [Structured outputs](../structured-outputs/README.md)
