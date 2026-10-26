# Images, PDFs and audio

Needs: [Tokens and tokenizers](../tokens-and-tokenizers/README.md) · [Context window](../context-window/README.md) · Reviewed October 2026

A model reads a picture, a PDF or a recording as tokens, and it gets only what the file was turned into on the way. A picture becomes image tokens or recognised text, a PDF its stored text or pictures of its pages, a recording a transcript or audio tokens.

- **Token**: a piece of text, often a word or part of one; the model reads and writes tokens, and requests are limited and billed in them.
- **Context window**: the most tokens a model can handle in one call, counting both what it reads and what it writes.
- **Image tokens**: the tokens a picture becomes, which the model reads beside the text.
- **Metadata**: what a file stores about itself beside its content, such as the date and place a phone's camera writes into a photo.
- **Text layer**: the characters a PDF stores for its text, the ones a viewer lets you select and a PDF library reads out.
- **Audio tokens**: what a recording becomes for a model that takes audio: the sound itself, turned into tokens with no text step between.

Who turns a file into tokens, and by whose rules:

- **A provider's API**: your code sends the file inside the request, by its address or by a file id, and the provider turns it into tokens by its own rules and bills them as input.
- **Your own server**: the model decides which files can go in: pictures, and with some models sound or video. A server takes no documents, so turning a PDF into text or pictures is your code's job. The model's image processor, the step that resizes a picture and cuts it into squares, sets how many tokens a picture becomes.
- **A ready-made app**: you attach a file and the app picks the way it reaches the model; one reads the pictures in a PDF of up to 100 pages and only the text of a longer one.

Each way works the same wherever it runs.

A file reaches the model in one of six ways:

| Way | What the model gets | Where you find it |
| --- | --- | --- |
| Image tokens | the pixels of a picture, or of a page image | a provider's API or your own server, with a model that takes images; a ready-made app |
| Text layer | the characters the PDF stores | a PDF library your code runs beside a provider's API or your own server; a ready-made app, for longer PDFs on one |
| Recognised text | the characters a recognition step, often called OCR, read off the pixels | a text-recognition tool your code runs beside a provider's API or your own server |
| Text layer and page images | the stored characters and a picture of each page | a provider's API that takes PDFs; a ready-made app, for shorter PDFs on one |
| Transcript | the text a speech-to-text model wrote down | a speech-to-text model on a provider's API or your own server |
| Audio tokens | the recording's sound | a provider's API or your own server, with a model that takes audio |

A file can also reach the model in pieces: a search tool cuts the file into chunks, finds the ones that match the question and adds only those to the request. A request with files meets the context window, counted in tokens, and often a separate size limit, counted in bytes.

Errors start where what you see and what the model received part ways: a clear scan, a sharp photo, a call you listened to yourself.

This topic lets you:

- send photos, screenshots, PDFs and recordings to a model knowing what reaches it and what each one adds to the bill
- choose how a scanned form, a paper questionnaire or a spoken answer should reach the model
- trace a wrong reply about a photo, a form or a voicemail back to what the model was given

###### 1. Re-saving listing photos at a lower quality

A property site has a model write a short description of each room photo that agents upload, through a provider's API. The photos are 1,200 × 800 pixels. To cut the bill, a teammate re-saves every photo at a lower JPEG quality before it is sent, and the files shrink to about a sixth of their size.

What happens to the number of input tokens a photo costs?

<details><summary>A. They drop to about a sixth, as the file size did</summary>

✗ The obvious length of a photo is its file size, the measure you would use for text. The model never reads the file's bytes. The provider decodes the file back into pixels and counts squares of them, and the re-saved photo has the same 1,200 × 800 pixels as before.

</details>

<details><summary>B. They fall to a small fixed amount per photo</summary>

✗ Some APIs do charge a small fixed amount per image, and a file this much smaller could qualify for it. That amount comes from an option you choose on the request, and the file's size never selects it. This photo is counted by its pixels, the same as before the re-save.

</details>

<details><summary>C. They drop somewhat, by less than the file size did</summary>

✗ If heavy compression throws away fine detail, a count of detail should drop with it. The count never looks at what the pixels show. A rule that cuts the picture into squares gets the same number of squares from a sharp 1,200 × 800 photo as from a blurred one.

</details>

<details><summary>D. They stay the same</summary>

✓ Right. Each photo bills exactly what it did before. The provider decodes the file into pixels and counts those, and the re-save changed the bytes, not the pixel size.

The finance lead sees the same cost per listing next month. The photos the model reads are rougher now: heavy compression blurs small text, and the energy label on an oven is harder to make out in every one of them.

Image tokens are made from the picture's pixels: the provider decodes whatever file you send and cuts the pixels into small squares. A published rule that counts squares of 28 pixels gives 1,247 for a 1,200 × 800 photo, and a rule that counts squares of 32 pixels gives 950, which some models then multiply. Neither takes the file size, the format or the quality as an input. Smaller files do help elsewhere: they travel faster and keep a request under its size limit. Where tokens are counted by squares, fewer pixels is what lowers the count, so crop each photo to the part that matters. Where an API lets a resolution option set the tokens per image, the option decides the count, and the file size plays no part there either. On your own server there is no bill, and the same pixels take the same room in the window.

</details>

---

###### 2. A claim form run through the office copier

An insurer's claims portal answers customers' questions about the documents they upload. Your code takes the text of each PDF page with a PDF library at its default settings and sends it with the question. A customer uploads a claim form they ran through the office copier as an ordinary scan, and in any viewer it looks as sharp as the paper.

What does the model get from the claim form?

<details><summary>A. The form's text, exactly as it reads on screen</summary>

✗ The form reads perfectly in any viewer, and anything you can read on a screen is natural to take for stored text. What the viewer draws is a picture of the paper. A PDF library reads only the characters a file stores, and an ordinary scan stores none, so each page yields no text at all.

</details>

<details><summary>B. The question alone, with no text from the form</summary>

✓ Right. Every page comes back from the library as empty text, with no error, and the request carries the customer's question with no form in front of it.

The customer asks what excess applies to their claim and gets a reply written from the question alone. The portal's log shows a clean upload and a successful call, and the claims handler who picks up the complaint finds no trace of the copier anywhere in it.

An ordinary scan saves each page as one picture of the paper, so the file has no text layer: no stored characters, only an image drawn across each page. A PDF library reads the text layer and returns what it holds, which here is nothing. Reading characters off pixels gives recognised text, the work of a separate step that the library does not run. A scan that went through that step before upload carries a text layer of recognised text, and the library returns it with any misreadings. Test every page: one that draws a picture and yields no characters goes to text recognition, or to a model that takes images, as a page image. An API that takes the PDF whole sends the text layer and page images together, so a scan does reach the model there, as pictures. A ready-made app picks the way for you, and one reads only the text of a long PDF.

</details>

<details><summary>C. Nonsense symbols where the form's words should be</summary>

✗ Garbled symbols are the usual failure when you copy text out of a PDF whose fonts map their letters strangely, and a page with no real letters at all would be the worst case of it. Nonsense needs stored characters to start from. The library does not read a picture's bytes as letters, and a page that stores no characters yields no text.

</details>

<details><summary>D. The form's text, with a few letters misread</summary>

✗ Your phone can pick the text out of a photo, and a library built for PDFs could be expected to do as much. Reading characters off pixels is text recognition, a separate tool that you run yourself. At its default settings a PDF library returns only stored characters, and this file has none to return.

</details>

---

###### 3. Grading pronunciation in a language app

A language-learning app has adults read a sentence aloud and tells them which sounds to work on. Today a speech-to-text model writes down each recording, and the text goes to a model along with the target sentence. The app calls almost every reading perfect, while the learners' teachers, marking the same sentences in class, keep finding vowels and stress to correct.

Which change lets the feedback point at the sounds a learner got wrong?

<details><summary>A. A more accurate speech-to-text model</summary>

✗ When transcripts go wrong, a better speech-to-text model is the usual fix. These learners say the right words, and the transcripts already write them down correctly. A more accurate model writes the same correct words more reliably, and the accent still never reaches the text the feedback model reads.

</details>

<details><summary>B. An instruction to judge the sounds from the transcript's spelling</summary>

✗ When a mispronounced word turns up as a wrong word in a transcript, a model can work back from the slip to the sound. The speech-to-text model writes real words in their usual spelling, the ones it judged the speaker meant. A vowel said wrong but still recognisable comes out spelled right, and the feedback model has nothing to judge.

</details>

<details><summary>C. The recording itself, given to a model that accepts audio</summary>

✓ Right. Only the setup that sends the recording gives the model the sound to judge. Every other change still hands it a line of text.

A learner whose vowels drift hears from the app that each sentence was perfect, books the speaking exam on the strength of it, and then sits in front of an examiner who hears what the teachers heard.

A transcript is text: the words the speech-to-text model settled on, spelled the usual way. An accent that leaves the words recognisable leaves no trace in it, and a learner with every word right and every vowel wrong produces the same line as a native speaker. Audio tokens carry the sound itself into the model, with no text step between, so stress, vowels and rhythm are there to be judged. Test the new setup on recordings a teacher has marked before you trust its feedback. Models that take audio are offered on some provider APIs, and some models you run on your own server take it too.

</details>

<details><summary>D. Higher-quality recordings, from a better microphone</summary>

✗ Better audio is the first thing a sound engineer reaches for, and a phone microphone is no studio. A cleaner recording helps the speech-to-text model hear the words, and it still writes down words. However good the recording, the feedback model receives the same line of text.

</details>

---

###### 4. Checking a moving estimate's photos against the request limit

A moving company estimates the volume of each move from photos of every room, sent through its app. Your code puts each photo into the request as base64, the usual way a JSON body carries a file's bytes, and `call_model` sends the request and returns the reply text. The provider's limit for one request is 32 MB, and your code checks the photos against that before sending. A customer with a large house sends 40 photos that come to 31.25 MB on disk, just under the limit, and your check passes them.

```python
import base64
import os

REQUEST_LIMIT = 32 * 1024 * 1024

def estimate_move(photo_paths, question):
    if sum(os.path.getsize(path) for path in photo_paths) > REQUEST_LIMIT:
        raise ValueError("Too many photos for one request")
    content = []
    for path in photo_paths:
        with open(path, "rb") as photo:
            data = base64.b64encode(photo.read()).decode()
        content.append({"type": "image", "data": data})
    content.append({"type": "text", "text": question})
    return call_model(messages=[{"role": "user", "content": content}])
```

What happens to this customer's request?

<details><summary>A. The request is refused for its size</summary>

✓ Right. The request is turned away before the model sees a single photo. Your check summed 31.25 MB, while the body that went out held the photos as base64 text, 41.67 MB of it, over the 32 MB limit.

The customers with the most rooms, the largest moves the company quotes for, are the ones shown a failure after your own check had cleared their photos. Every batch from about 24 MB up to the limit fails this way, and a test with a handful of photos never gets near it.

Base64 spends 4 characters on every 3 bytes, so each photo is a third larger inside the request than on disk, plus a little for the JSON that wraps it. What the limit measures is the body your code sends, not the files it read. Measure the body after encoding, or hold the raw total under about three quarters of the limit, 24 MB here. For many photos or large ones, upload each photo to the provider once and pass the file id it returns, which keeps the body small. The context window, counted in tokens, is a second limit the same photos must fit as well.

</details>

<details><summary>B. The request is accepted, and the customer gets an estimate</summary>

✗ Base64 is an encoding of the same bytes, and nothing in the word encoding warns you of growth. Each base64 character holds only 6 bits, which makes 4 characters out of every 3 bytes. The 40 photos go out as 41.67 MB of text, and the 32 MB limit is measured on that body.

</details>

<details><summary>C. The request is accepted, with the photos scaled down to fit</summary>

✗ Scaling down an oversized picture is something some providers do. Scaling acts on a picture's pixels once the request has been taken in and decoded. The size limit is applied before that, to the request body, and a body over it is turned away whole.

</details>

<details><summary>D. The request is refused for holding too many tokens</summary>

✗ Paste a base64 string into a text prompt and it is read as words, costing tokens like any other text, so the model's window looks like the limit to hit. Here the string stands for a photo's bytes in an image entry, and the model reads image tokens made from the pixels they decode to. The size limit acts first, on the body's bytes: 41.67 MB against 32 MB, turned away before anything is decoded or counted.

</details>

---

###### 5. Checking when a damage photo was taken

A car insurer has a model check, through a provider's API, whether each damage photo looks recent, to catch old photos sent with new claims. The photos arrive straight from customers' phones, and each file still holds the date and place that the phone's camera wrote into it.

What does the model have to go on when it judges a photo's age?

<details><summary>A. The whole file, its stored date and place included</summary>

✗ Your phone's gallery prints the date and the place under every photo, straight from the file you are sending. The gallery reads the file's metadata, while the model reads image tokens made from the pixels alone, and the date and place stay behind in the file.

</details>

<details><summary>B. The camera's date, with the place removed</summary>

✗ Photos you have shared sometimes lost their location and kept their date, a privacy habit you might credit to a provider. Nothing is filtered field by field here. The model receives none of the file's metadata, the date included, only the picture's pixels.

</details>

<details><summary>C. The picture alone</summary>

✓ Right. The model gets the picture and nothing else from the file. The date and the place stay in the file's metadata, and none of it reaches the model.

A photo of a dented door taken two winters ago looks as fresh as one taken this morning, and the model can judge only from what the picture shows. An old photo sent with a new claim passes as recent, and the insurer pays out on damage that may be years old.

Image tokens come from the picture's pixels, and the camera's date and place are not pixels: they sit in a separate block of the file. Decoding the file into pixels leaves that block behind, and the model receives none of it. If the date matters, read it from the file in your own code and check it there or send it as text, and treat it as a claim, since camera data can be edited. A server you run yourself decodes the file into pixels in the same way.

</details>

<details><summary>D. The picture, plus the date and place written out as text</summary>

✗ A PDF's stored text does go to the model beside its page images on some APIs, which makes a photo's stored fields riding along as text a fair guess. A picture is decoded to its pixels, and those are what the provider passes on. The camera's fields are not written out for the model, as text or in any other form.

</details>

---

###### 6. Asking about a photo by its file name

A renovation firm lets its site managers upload photos of a job to an assistant that answers questions about them. Your code reads the twelve photos of one job from the job's folder and puts their bytes into a single request to a provider's API, followed by the question the manager typed: "Is there damp on the ceiling in garage.jpg?"

What does the request give the model to tell which photo is garage.jpg?

<details><summary>A. Each picture tagged with its own file name</summary>

✗ Every file manager, mail client and upload form shows the name beside the file, so the name seems to travel wherever the file goes. It is a label that the folder keeps outside the file's bytes. The request holds the pictures' pixels and the text your code writes, and a name is neither unless your code writes it in.

</details>

<details><summary>B. Twelve pictures in order, with no names</summary>

✓ Right. The model gets twelve pictures in the order your code put them, then the question as text, and no picture carries a name.

A manager reads an answer about a ceiling and takes it for the garage's. If it describes the kitchen, the surveyor sent out to look at damp goes to the wrong room, and the photo called garage.jpg was never the one described.

A file name lives in the folder, not in the file: it is a label kept beside the bytes, so reading the bytes into the request never picks it up. The request holds the pictures, which the model reads as image tokens, and the text your code writes, and nothing else. To make the name usable, write it into the text yourself: put a line such as "Photo 3: garage.jpg" before each picture and let the question use those labels. A server you run yourself works the same way, since the model there reads only what the request holds.

</details>

<details><summary>C. One list of all twelve names, in the order of the photos</summary>

✗ Anyone who has listed the files in a provider's file store has seen the names in a column, and expects the same column to reach the model. Here the photos travel inside the request, and a request holds only what your code put in it. It carries no list of names unless your code writes one.

</details>

<details><summary>D. An error for naming a file the request does not hold</summary>

✗ Tell a shell to open a file that is not there and it stops with an error, so a question that names "garage.jpg" invites the same expectation. The manager's question is plain text to the model, and nothing checks it against file names. The request goes through as twelve pictures and a sentence that mentions one of them by a name.

</details>

---

###### 7. Shrinking plant photos before upload

A garden centre runs an app that names plants from customers' phone photos, through a provider's API. The phones save each photo's pixels exactly as the camera's sensor read them, and every original looks upright in the phone's gallery. To speed up uploads, your code shrinks each photo to 1,000 pixels on its long side. Its image library builds the smaller copy from those stored pixels alone and keeps none of the metadata. A customer holds the phone upright to photograph a fern.

Which way up is the fern in the picture the model gets?

<details><summary>A. Lying on its side</summary>

✓ Right. The copy reaches the model on its side. Its pixels are the ones the phone's sensor read, in that order, and the copy holds nothing that says to turn them.

A customer who photographs a fern upright sees it fine in the gallery, while the copy in the app's upload folder lies on its side, and so does the picture the model is given. The adviser who opens that folder to check a bad identification finds the upright shots all lying down.

A phone's sensor reads every picture wide, as if the phone were held sideways. A phone that saves the pixels as the sensor read them stores a photo taken upright lying on its side, with a note in the file's metadata that tells a viewer to give it a quarter turn. Galleries and browsers apply the note, which is why every original looks upright. Your resize built a new picture from the pixels and left the note behind, so the copy has sideways pixels and no instruction. The model reads image tokens made from those pixels. Turn each photo upright by its note in your own code before shrinking and saving it, and open a few saved copies in a plain viewer before you ship. A server you run yourself is given the same sideways pixels.

</details>

<details><summary>B. Upright, as in the gallery</summary>

✗ The gallery shows the original upright, and the copy is a smaller version of the same photo, so it ought to be upright too. The gallery is applying a rotation note stored in the original's metadata, and the copy was saved without that metadata. What is left is the sensor's pixels, on their side.

</details>

<details><summary>C. Upside down</summary>

✗ When a photo is turned the wrong way, upside down is the picture you reach for first, like a book held the wrong way up. The sensor reads in landscape, so a photo taken with the phone upright is stored a quarter turn from upright, not a half turn, and without the note the copy lies on its side.

</details>

<details><summary>D. No picture: the request fails with an error</summary>

✗ A form refuses a submission with a field missing, and stripped metadata is a field missing from the file. A JPEG with no camera data is a complete, valid file, and the model takes its pixels like any other photo's. All it lacks is the note on how to turn the picture.

</details>

---

###### 8. Setting the low-detail option for recipe cards

A recipe app turns photos of handwritten family recipe cards into typed recipes, through a provider's API. The photos are 3,000 × 2,000 pixels. To cut the bill, you set the API's low-detail option on every image, which charges each one a small fixed number of tokens whatever its size.

What does the model get from each card?

<details><summary>A. The same full-size card, at a discount price</summary>

✗ A cheaper option that leaves the work unchanged is what a discount tier means, and this option is named like one. It changes the picture, not the price list. A handful of tokens covers only a small copy of the card, and that copy is all the model reads.

</details>

<details><summary>B. A small copy of the whole card</summary>

✓ Right. The model gets a much smaller copy of the card, scaled down to fit the few tokens the option allows.

The handwriting reaches the model a few pixels tall. Fractions and notes squeezed into the margin are the first things to blur away, and the person cooking from the typed recipe cannot see that the card said more.

Image tokens are made from squares of the picture's pixels, so a fixed, small token count covers only as many squares, and the picture has to shrink to fit them. Picture an option that allows 256 squares of 28 pixels: a 3,000 × 2,000 card that would need 7,776 squares arrives as about 532 × 355 pixels, and handwriting 40 pixels tall shrinks to about 7. The saving is real, and it is paid for in detail. Keep the low option for pictures where the gist is enough, such as sorting photos by room, and send writing the model must read at a higher setting, or cropped. On your own server the same trade is the image processor's largest size, a setting you choose.

</details>

<details><summary>C. The same full-size card, read in a quick skim</summary>

✗ For a person, a quick glance is the cheap way to read a page, and a setting called low detail hints at a skim. The model has no setting for attention on an image. It reads the image tokens it gets, and a fixed handful of them can only describe a small copy of the card.

</details>

<details><summary>D. The middle of the card, cut to a fixed size</summary>

✗ Every avatar upload you have done cut a fixed square from the middle of the photo. The option keeps the whole card and shrinks it to fit. The edges arrive along with the middle, all of it at a fraction of the original detail.

</details>

---

###### 9. Customers attaching screenshots of an error

A web-hosting company runs a help chat that passes each customer's message to a model on a provider's API. Many customers attach a screenshot of their browser window, 1,280 × 720 pixels, where the page shows one line, "Error 504: gateway timed out after 30 seconds", that they could have pasted as text.

How does the screenshot's input cost compare with the same line pasted as text?

<details><summary>A. The same as the pasted line, token for token</summary>

✗ The model ends up with the same words either way: it reads the error off the picture. What it is billed for is the picture. The 1,280 × 720 window is cut into squares and every square counted, 1,196 tokens under one published rule, against about 11 for the line as text.

</details>

<details><summary>B. Many times what the line costs</summary>

✓ Right. The screenshot costs many times what the pasted line would: 1,196 image tokens under a published rule of 28-pixel squares, and 920 under a rule of 32-pixel squares before any multiplier, against about 11 tokens for the line.

Most chats on a busy morning open with a screenshot, and the support manager watches the bill for the help chat grow while the conversations stay as short as ever. Each screenshot costs 109 times the typed line under one published rule, and 84 times under the other before any multiplier.

Image tokens come from the picture's pixels, not from the words it shows: the provider cuts the whole 1,280 × 720 window into squares, the blank page around the error included, and counts them. The words drawn in the picture add nothing to the count and save nothing, and a window full of dense text would cost the same. Ask customers to paste error messages as text, or crop the screenshot to the error before sending, since a smaller picture covers fewer squares. Where an API offers a low-detail option the count drops, at the price of a smaller copy that can blur the text. On your own server there is no bill, and the same screenshot takes the same room in the window.

</details>

<details><summary>C. Less than the pasted line</summary>

✗ The chat shows the screenshot as a single attachment beside a whole line of words, and one item sounds like the smaller bill. The picture is billed by its pixels: a mostly empty browser window comes to 1,196 squares under one published rule, while the line it shows takes about 11 tokens as text.

</details>

<details><summary>D. Slightly more than the pasted line</summary>

✗ You may expect the provider to read the text off the picture and bill those words, with something added for the image. Billing never looks at what the picture says. The count goes by squares of pixels across the whole window, blank space included.

</details>

---

###### 10. Asking the model to zoom in on a shelf photo

A wine shop has a chat app, built on a provider's API, that lets customers photograph a shelf and ask about a bottle; the model is given no tools. The app shrinks each photo to 1,024 × 768 pixels before sending, and your code resends the chat history on every turn. The model replies that the label on the bottom-left bottle is too small to read, and the customer types: zoom in on that bottle.

On that turn, what is there for the model to read the label from?

<details><summary>A. An enlarged crop of that bottle</summary>

✗ Zoom is two fingers away in every photo app you use. The model has no picture to enlarge beyond the tokens in the request. Those were made from 1,024 × 768 pixels on the first turn and are made from the same pixels now; the words "zoom in" arrive as text and add no detail.

</details>

<details><summary>B. The whole photo resent at a higher detail setting</summary>

✗ Detail is a setting some APIs take with each picture, the obvious thing for a closer look to raise. That setting is a field your code writes on the request, and the customer's words never change it. Your code sent the photo exactly as before, at the same setting and the same size.

</details>

<details><summary>C. An enhanced copy with the label made sharp</summary>

✗ Phone galleries offer a one-tap enhance that turns a soft photo crisp, and a service built around a model is where you would expect one. Nothing in the request asks for it, and no step can put back detail the shrunken copy never held: a label too small to read in 1,024 × 768 pixels stays too small in any copy made from those pixels.

</details>

<details><summary>D. The same picture as before</summary>

✓ Right. The request carries the same 1,024 × 768 photo as before, the same 1,036 image tokens under a published rule of 28-pixel squares, and one new line of text asking for a zoom.

Whatever the next reply says about the label rests on the very pixels that were too small a turn earlier. A customer choosing between two vintages of the same wine can take that reply at its word and walk out with the wrong bottle.

Every image token the model reads was built from the pixels the request carries, and the model has no way to fetch more: with no tools, it reads the request and writes text, and "zoom in" reaches it as words. A closer look has to come from your code. Crop the region around the bottle from the phone's full-size photo, 4,032 × 3,024 pixels here, and send the crop as a new picture: at the original's scale the label stands about four times as tall as in the shrunken copy, and it spreads across many more squares. A server you run yourself works the same way, since the model there also sees only the pixels your code sends.

</details>

---

###### 11. Reading circled answers on paper questionnaires

A polling firm scans paper questionnaires and has a model record which of five answers each respondent circled. The five answers sit side by side, a few millimetres apart, and spot checks keep finding the model picking a neighbour of the circled one. Before the next survey goes to print, you can change one thing.

Which one change should most cut the misread answers on the next survey?

<details><summary>A. Scan the forms at twice the resolution</summary>

✗ Your first fix for a misread picture is usually more pixels, and the scanner has a higher setting waiting. A 2025 study that tested this kind of mark found models struggling with shapes that overlap or sit close together at every image resolution it tried. The circle is visible already; what fails is telling which answer it surrounds.

</details>

<details><summary>B. Have the model reason step by step before it answers</summary>

✗ Step-by-step reasoning is the cheapest change on the list, the one that rescues maths and logic problems. A 2025 study of simple marks also tested models that reason at length before answering, and they struggled the same way with shapes that sit close together. The trouble lies in reading the picture, and more reasoning on top of a misread does not repair it.

</details>

<details><summary>C. Move to the most capable model on offer</summary>

✗ A bigger, newer model is the usual answer to a hard task, and switching models is a change to one setting. In a 2025 study of such marks, the best of four leading models was right on about 78% of its simple tasks, while spacing the shapes apart brought accuracy close to 100%. The page layout moved the result further than the choice of model.

</details>

<details><summary>D. Print the five answers farther apart</summary>

✓ Right. Spacing the answers apart is the change a study of such marks found to work: models that misread closely packed shapes read them almost perfectly once more space separated them.

Keep the old layout and each misread mark moves a respondent one place along the scale, from agree to strongly agree. The client reads the shift as a change of opinion and presents it to its board as a trend.

Image tokens can hold the circle and all five answers, and a model can still fail to say which answer the circle surrounds. In its 2025 version, the study Vision language models are blind tested leading models on simple pictures, among them telling which letter in a word has a circle drawn around it. They struggled when shapes overlapped or sat close together, at every image resolution tried, and models that reason at length before answering struggled too; the best of the four was right about 78% of the time across the set. With more space between the shapes, accuracy rose to near 100%. Those were models of 2024 and 2025, so measure yours on a sample of real forms. Space the answers apart, give each its own box, and keep the spot checks going.

</details>

---

###### 12. Blacking out a tenant's name before upload

A tenants' advice charity has an assistant that summarises the letters advisers upload. Your code runs each letter through a PDF library's default text extraction and sends the result to a model. Before uploading a landlord's letter, an adviser draws filled black rectangles over the tenant's name and address with a PDF editor's shape tool and saves the file. On screen and on paper, both are gone.

What reaches the model from the lines under the rectangles?

<details><summary>A. The tenant's name and address</summary>

✓ Right. The name and address arrive as ordinary text among the rest of the letter, with nothing to show that anyone covered them.

The adviser believes the tenant's details never left the office. They went out to the provider inside the request, into any log your code keeps of its requests, and nothing stops the summary from opening with the tenant's name, the one detail the adviser meant to keep back.

A PDF keeps its text layer and its drawings as separate instructions. The letter's characters are stored as text, and the rectangle is a shape painted after them: a viewer draws it over the letters, while text extraction takes every character the page stores and pays no attention to what is painted on top. Whether the editor writes the rectangle into the page or lays it over the page as a separate note, the characters stay in the file. To take them out, use a PDF tool that deletes the text under each box, or turn every page into a page image; either way, extract the text yourself before upload and check that the name is gone. An API that takes the PDF whole reads the same text layer beside its page images, and so does a ready-made app that reads a PDF's text, so the name reaches the model on every route that reads the text layer. Only a model given the page image alone sees the black bars.

</details>

<details><summary>B. Nothing from the covered lines</summary>

✗ On screen and on paper the name is gone, the only two places the adviser checked. The bars are shapes painted over the letters. The characters underneath are still stored in the text layer, and extraction hands them over with the rest of the page.

</details>

<details><summary>C. A placeholder where the name stood</summary>

✗ Tools that strip personal data from text leave a marker in its place, such as [name]. The shape tool strips nothing and only adds a drawing, so the stored characters stay as they were. Extraction has no marker to write and returns the name itself.

</details>

<details><summary>D. The name and address, flagged as hidden</summary>

✗ A program that can read the text under a box would surely know the box was there. Extraction takes the characters a page stores and ignores what is drawn over them. The covered name comes back with the same standing as every other line of the letter, with no flag or note beside it.

</details>

---

###### 13. A price list with empty pallet prices

A builders' merchant answers trade customers' price questions from suppliers' PDF price lists. Your code pulls the text of each list with a PDF library's default extraction, which writes this list out as plain text, one table row to a line, and sends it with the customer's question. One supplier's list is a table with three columns, the product, the price per bag and the price per pallet, and products sold only by the bag have the pallet cell left empty.

What does the model get for a product sold only by the bag?

<details><summary>A. The row, with an empty slot under the pallet price</summary>

✗ Copy a table out of a spreadsheet and the empty cells come with it, as tab stops in the right places. A PDF keeps nothing of the sort. Its text layer holds characters at positions, an empty cell holds none, and the library writes out the characters it finds, leaving no slot where nothing was.

</details>

<details><summary>B. The product, with the pallet price given as zero</summary>

✗ In a spreadsheet formula an empty cell counts as zero. A PDF library does no arithmetic and fills in nothing. It writes out the characters the page stores, an empty cell stores none, and no zero appears: the pallet price is absent from the line.

</details>

<details><summary>C. Each price, labelled with its column heading</summary>

✗ The column headings sit right above every price, plain to anyone reading the page. The library writes the heading row once, as one more line of text, and each price after it as a bare number. Only its position tied a number to the heading above it, and the text written out keeps no positions.

</details>

<details><summary>D. The product name and one price, nothing more</summary>

✓ Right. The line reads as the product name followed by one price, and nothing on it marks the empty cell or says which column the price came from.

A product sold only by the pallet comes out in the same shape, a name and one number. A plasterer who asks for the pallet price of building sand can be quoted the bag price as if it were per pallet, and the merchant then either honours forty pallets at that figure or argues with a trade customer.

A PDF holds no table, only characters placed at positions on the page, with the grid lines drawn as shapes. Here extraction writes the characters out one row to a line; the gap where a cell is empty holds no character, so it leaves nothing behind. The model gets the text layer with the grid gone: a row with both prices ends in two numbers, a row with one price ends in one, and nothing says which column that one came from. Use the library's layout mode if it has one, an extraction setting that keeps the horizontal gaps as spaces, or a tool built to read tables, and turn each row into labelled fields before sending; or send the page as a page image, where the columns stay visible. An API that takes the PDF whole sends page images beside the text, so the columns are at least in front of the model there.

</details>

---

###### 14. Transcribing voicemails with long pauses

A doctors' surgery records patients' voicemails after hours. A speech-to-text model writes each message down, and a model drafts a note for the morning receptionist from the transcript. Many callers stop for long stretches while they look for a letter or a prescription.

What can the transcript of such a message hold that the caller never said?

<details><summary>A. Whole phrases and sentences</summary>

✓ Right. A transcript can hold whole phrases or sentences that are not in the recording, and they turn up more often for callers who pause at length.

A note can tell the receptionist that a caller asked for a repeat of a medicine they never named. The receptionist, with forty messages to clear before the doors open, acts on the note and not on the recording.

A transcript is the speech-to-text model's text, and the model that drafts the note takes it as the call. A 2024 study of a widely used speech-to-text model found that roughly 1% of its transcripts held phrases or sentences no one had said, and more of them for speakers with longer silent stretches. The model it tested was the 2023 version of that service, so measure the one you use. Keep the recording one click from every note, and have staff check names, medicines and dates against it before acting.

</details>

<details><summary>B. No extra words, only gaps where spoken words dropped out</summary>

✗ Automatic captions you have watched drop a mumbled phrase now and then, and gaps are the error you have learned to expect. Gaps do happen, but a transcript can also gain text. In a 2024 study, roughly 1% of one widely used model's transcripts carried phrases or sentences nobody spoke, more often around long silences.

</details>

<details><summary>C. A mark where each pause fell</summary>

✗ Written records of speech, from court transcripts to subtitles, often note a pause. A speech-to-text model is asked for the words, and silence is not something it reports. The risk around long silences runs the other way: text that was never spoken.

</details>

<details><summary>D. A wrong word here and there in place of a spoken one, and no more than that</summary>

✗ Speech-to-text has grown good enough for meetings and captions that a stray misheard word is the worst you expect. Accuracy on clear speech says little about long silences. A study published in 2024 caught a widely used model writing whole sentences into about one transcript in a hundred, more often for callers who paused at length.

</details>

---

###### 15. Hold music on recorded support calls

A broadband company records every call to its support line. When a call ends, your code sends the recording as an audio file through a provider's API, to a model that accepts sound, with a request for a summary. Many calls open with four minutes of hold music before the caller is put through.

What do the four minutes of hold music add to a call's input tokens?

<details><summary>A. No tokens at all</summary>

✗ Tokens are how text gets counted, and hold music has no words in it to count. Audio tokens are made from the sound, not from words, and they are counted by the second. Four minutes of music bill exactly what four minutes of a caller talking would.

</details>

<details><summary>B. A fraction of what four minutes of speech would add</summary>

✗ An audio codec spends almost nothing on silence or steady music, as any file size shows. The count never asks how well a stretch compresses. Audio tokens go by time, and four minutes of music cost what four minutes of anything cost.

</details>

<details><summary>C. As much as four minutes of speech</summary>

✓ Right. The music bills like any other four minutes of sound: audio tokens are counted by the length of the recording, whatever is in it.

On the busiest days the queue is longest, and every summary carries the longest stretch of music, so the audio bill climbs with the wait and not with the faults reported. The operations manager watches the cost per call rise in the very weeks the line is struggling.

Audio tokens carry the sound itself into the model, and providers count them by the second. Two published rates, 25 tokens a second and one token for every 100 milliseconds, turn the four minutes into 6,000 and 2,400 tokens. Neither counts words, the sample rate or the file's size, and a provider that documents its processing first reduces every recording to a single channel at a low bit rate. Cut the hold music and long silences from the file before you send it. A live voice session that listens for speech can leave empty stretches out of the count, while a finished recording sent as a file is counted whole.

</details>

<details><summary>D. More on a studio-quality file than on a phone-line recording</summary>

✗ A studio file holds far more sound than a phone line, and more sound would mean more for the model to read. The count goes by length alone: before counting, a provider that documents its processing reduces every file to one channel at a low bit rate, and per-second rates ignore quality, so four minutes from either source bill the same.

</details>

---

## Further reading

- [Careless Whisper](https://arxiv.org/abs/2402.08021), 2024: roughly one transcript in a hundred from a widely used speech-to-text model held phrases or sentences never spoken, more often for speakers with long silences.
- [Vision language models are blind](https://arxiv.org/abs/2407.06581), 2025: leading models misread simple pictures, such as which letter in a word is circled, until the shapes are spaced apart.
- [DocVQA](https://arxiv.org/abs/2007.00398), 2020: 50,000 questions on more than 12,000 document images, where models trailed people most on questions that rest on a page's structure.

Next: [Embeddings and vector search](../embeddings-and-vector-search/README.md)
