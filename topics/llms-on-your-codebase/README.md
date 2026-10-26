# LLMs on your codebase

Needs: [Generation parameters](../generation-parameters/README.md) · [Retrieval and RAG](../retrieval-and-rag/README.md) · [Agents and workflows](../agents-and-workflows/README.md) · [Security](../security/README.md) · Reviewed October 2026

A model works on a repository through a coding assistant in your terminal or editor, a review bot on each change, or an agent that edits code and runs commands by itself. Code travels both ways: from the repository into each request, and from the model's reply back into files, commands and tests.

- **Token**: a piece of text, often a word or part of one; requests and replies are counted in tokens.
- **Tool call**: the part of a model's reply that asks for a tool, with the tool's name and the arguments; the code or the app that runs the loop runs the tool and sends back the result.
- **Chunk**: a piece of a file, cut out ahead of time and stored in a search index, which a search can find and paste into a request.
- **Hunk**: one piece of a diff: the changed lines with unchanged lines of context above and below them, under a header, such as `@@ -13,7 +13,7 @@`, that gives the line where the hunk starts in the old file and in the new one, and how many lines it spans in each.
- **Generation cap**: the request's limit on how many tokens the reply may run to, often called max tokens.
- **Sandbox**: whatever you run an agent's commands inside to limit what they can reach.

When you build the bot or the agent yourself, on a provider's API or your own server, your code selects the code, applies the patches and runs the commands and tests, so every limit on what is sent and what runs lives in your code; with your own server, the code, secrets and services those commands can reach also sit on machines you run. A ready-made app, such as a coding assistant in your terminal or editor or a hosted review bot, runs its own loop, picking files, proposing edits and sometimes running commands, and you set what it may open and do. The model works from the code it is sent the same way everywhere.

Your system gets code to the model in four main ways:

| Way | What it sends |
| --- | --- |
| Search by the model | the lines that match each text search the model asks for, one tool call at a time |
| Chunked search | pieces of files that a search over an index picks for the question |
| Symbol index | the definition of a name, looked up in a table of names with the file and line where they are defined |
| Whole files | the full text of each file that your system picks, that you point it at, or that the model asks to read |

A model can name a library or an import that nobody has published. An agent allowed to run a test or build command without asking runs whatever the files behind that command say (see AI system design).

This topic lets you:

- judge which code a tool puts in front of the model, from its own searches to whole files, and ask it whether a function is still used
- weigh what a passing test run or a coverage figure (the share of code lines the tests ran) proves about a change an agent made, and choose a model off a public leaderboard
- set what an agent's commands and the packages it adds can reach, and check its edits before they land on your branch

###### 1. Asking a coding assistant whether a function is still used

Before deleting an old helper, `legacy_refund`, a developer on a food-delivery app asks the coding assistant in their terminal whether anything still calls it. The assistant searches the whole repository for the text `legacy_refund(`, gets back one line, the helper's own definition, and answers that nothing calls it.

Which code could still be calling `legacy_refund`?

<details><summary>A. Any code that reaches it without that exact text</summary>

✓ Right. The search established only that `legacy_refund(` appears on one line of the repository, the definition. Code that reaches the helper without writing that text is still a possible caller, and the model's "nothing calls it" rests on that single line.

Suppose the nightly refund job finds its handler through a jobs file that reads `handler: legacy_refund`, with no bracket after the name. The developer deletes the helper before lunch, the job fails at two in the morning, and the day's refunds sit unpaid until the support lead reads the failed run.

Here the code reaches the model through search by the model: the model asks for a text search as a tool call, the assistant runs it over the files on disk, and the matching lines come back as the tool's result in the next request. The search program read every file; the model read one line. A text search answers whether a string appears, and code can reach a function without that string: a name in a settings file, a function stored in a table and called through it, a name put together at run time, another service that calls it over the network. Before a delete, search for the bare name as well as the call, look for the places your code keeps function names as data, and read "no match" as a statement about text.

</details>

<details><summary>B. No code at all: the helper is safe to delete</summary>

✗ A search across every file is the check you would run yourself before a delete, and it found the definition, so it worked. It looked for one string, the name followed by a bracket. Code that reaches the helper through a string, a table of handlers or a name built at run time holds no such text, and the search passes it by.

</details>

<details><summary>C. Only code that builds its name at run time</summary>

✗ A name put together at run time is the textbook way past a text search, so it is the gap you remember, but it is only one of several. A jobs file that reads `handler: legacy_refund`, or a table that maps an event to the helper, names it with no bracket after it, and both run the helper while the search for the call finds neither.

</details>

<details><summary>D. Only other services that call it over the network</summary>

✗ Another service is the caller a repository search is known to miss, so the rest of the repository feels settled. The search was narrower than that: it matched the name followed by a bracket. A scheduled job named in a settings file, or a table of handlers, reaches the helper from inside the repository with no such text.

</details>

---

###### 2. Asking a code search why a bill came out lower

Developers at an electricity supplier ask an internal assistant about the billing service. The service's code is split into chunks, one per function, and the assistant answers each question from one search over those chunks. A developer asks why bills for customers in Wales come out 5% lower. The top chunk is `bill_total`, whose last line calls `apply_regional_rate`, a function defined in another module.

What puts the code of `apply_regional_rate` into the request the model answers from?

<details><summary>A. The call to it on the last line of <code>bill_total</code></summary>

✗ In an editor one click on a call opens the definition, and that habit turns the call into a link. To the search it is a few words inside `bill_total`'s chunk. Each chunk is ranked on its own text, and nothing in the ranking follows a name to the place where it is defined.

</details>

<details><summary>B. The import of it at the top of <code>bill_total</code>'s file</summary>

✗ The import line is where a file says where the helper comes from, so it reads as part of the context. Once the file is split function by function, its import lines fall outside `bill_total`'s chunk, and even an import carried along would name a module, not the code inside it.

</details>

<details><summary>C. Its own chunk, picked by the search next to <code>bill_total</code>'s</summary>

✓ Right. Only its own chunk ranking for the question puts the helper's code in the request. The call in `bill_total` carries the name and the arguments, `apply_regional_rate(subtotal, region)`, and nothing of what the function does.

When the helper's chunk misses the cut, the request holds `bill_total` and one line that names the helper, and whatever the model says about the 5% rests on that name. The billing analyst who passes the explanation on to a customer in Cardiff is passing on a guess about a rate table nobody sent.

A chunked search ranks each chunk on its own text against the question. A call is a few words inside the caller's chunk, and the search does not follow it to the chunk that defines the function; an import line does not either, and once the file is split at its functions, the imports belong to no function's chunk. With one search per question, the request is all the model gets. Code is a web of calls, and a question about behaviour often needs the callee more than the caller. After the search, look up each function the top chunks call in a symbol index and add its definition to the request, or give the model search tool calls so it can open the helper itself.

</details>

<details><summary>D. A link the index keeps from <code>bill_total</code>'s chunk to the function it calls</summary>

✗ Tools that map a repository's calls exist, so an index that links each chunk to the functions it calls is easy to picture. This assistant answers from one search, and a search ranks each chunk on its own text: nothing in the ranking follows a link from the top chunk to another. The callee's code reaches the request only when its own chunk ranks.

</details>

---

###### 3. Indexing names with a pattern

You build the coding assistant for a travel-booking company's developers, and it resolves a name through an index of symbols your code builds: it scans each source line for a function definition and records the name with its file and line. One module defines `fetch_user` at the top level with `async def fetch_user(session, id)`, a `PriceSheet` class with a `total` method, and a `slugify` created with `slugify = ...` instead of `def`. A developer asks the assistant to explain `fetch_user`.

```python
index = {}
for path in source_files:
    for lineno, line in enumerate(read(path).splitlines(), start=1):
        found = re.match(r"def (\w+)\(", line)
        if found:
            index[found.group(1)] = (path, lineno)
```

What does the lookup return for `fetch_user`?

<details><summary>A. Its file and line</summary>

✗ The scan visited every line of every file, so it is tempting to assume every definition landed in the index. It keeps only lines that begin with `def ` and a name. `fetch_user` is written `async def`, which the pattern never matches, so its entry was never made.

</details>

<details><summary>B. Nothing: the name is not in the index</summary>

✓ Right. The symbol index never recorded `fetch_user`, so the lookup misses, and the assistant explains the function with no idea where it lives, or reports that it cannot find it.

A developer asks about the one function that drives logins, and the assistant, missing it from the index, pulls in unrelated code or guesses the behaviour from the name.

The pattern matches a line that begins `def name(`. Here `async def fetch_user` begins with `async`, `total` is indented inside its class, and `slugify =` has no `def` at all, so the index holds the module's plain top-level functions and none of these three. A pattern over lines catches only the shapes it was written for, and real code names things many ways: async functions, methods, decorated functions, assignments, re-exports. Each shape it misses is a name the lookup cannot find, and the gap is silent, since a miss looks the same as a name that is genuinely absent. Build the index from a parse of the code, which knows every definition and where it sits, rather than from a line pattern.

</details>

<details><summary>C. The line of the class that contains it</summary>

✗ `fetch_user` is a top-level function here, not a method, but even for a method the index would hold names, not the classes around them. The scan stores one name against one line whenever a line matches, and never records what contains what, so there is no class for it to return.

</details>

<details><summary>D. The file and line of the function directly above it</summary>

✗ A failed lookup could fall back to the closest match above, the way an editor jumps to a definition. This index is a plain table from a name to one location: a miss is a miss, with no nearest-neighbour guess. The name is absent, so the lookup hands back nothing.

</details>

---

###### 4. Building an agent's context from the project folder

You build an agent that fixes bugs in a clinic's booking app, and it gathers its context from the project folder. The folder holds `app.py`, `utils.py`, a `README.md`, a `.gitignore` that lists `.env`, and `.env` itself, which holds the production database password.

```python
def gather_context(folder):
    files = []
    for path in folder.rglob("*"):
        if path.is_file():
            files.append(read(path))
    return files
```

Which files go into the request?

<details><summary>A. The tracked files only; the ignore list keeps <code>.env</code> out</summary>

✗ Your tools skip a file in `.gitignore` every day, so you count on it being left out everywhere. That list tells version control which files to leave untracked. It is not a filter on your own walk: `gather_context` reads whatever is on disk, and `.env` is on disk.

</details>

<details><summary>B. Every file but <code>.env</code> and <code>.gitignore</code>, the two hidden ones</summary>

✗ A leading dot marks a file as hidden, and anything hidden invites you to treat it as skipped. That only keeps the file out of a plain shell listing, nothing more. `rglob("*")` walks every file under the folder, dot or not, so `.env` and `.gitignore` are both read.

</details>

<details><summary>C. Only the source files, <code>app.py</code> and <code>utils.py</code></summary>

✗ Picturing the walk as picking code is natural for an agent that fixes bugs. This one has no such filter: `gather_context` keeps every path that `is_file()` accepts, and the README, `.gitignore` and `.env` are files. Keeping only source files is a rule you would have to write, such as an allowlist of extensions.

</details>

<details><summary>D. All five files, <code>.env</code> and its password included</summary>

✓ Right. The walk collects all five files, and `.env` goes into the request with the production password in plain text.

The password now travels with the request: to wherever the model runs, into any copy the provider keeps under its terms, and into the agent's saved session. The on-call engineer has to rotate it before the morning's first appointments open.

Sending whole files means choosing them, and `rglob` chooses everything: it walks the filesystem and returns every file; `.gitignore` is a list that version control reads, not a filter on your code, and no flag on disk marks a file as off-limits. Everything the model reads arrives in a request, so a file your code collects is a file you have sent (see Data and privacy for where the copies then live). Select by an allowlist of the extensions and folders the agent needs, read and apply the ignore list yourself if you walk the tree, and keep secrets out of any folder the agent can reach. On your own server the file would stay on hardware you run, but it would still be in the request the model reads.

</details>

---

###### 5. A review bot that reads the diff

A software team runs a review bot on its pull requests: for each one it sends the model the change's diff, made with your version control's default settings, and asks whether the change is safe. A developer edits one line deep inside a forty-line pricing function, a shipping-fee multiplier, and the bot approves.

What does the bot's approval tell you about the change?

<details><summary>A. That the whole forty-line function still works with the new multiplier</summary>

✗ The change is one line of a function, and the function is the unit you would open to judge it. A diff is built from hunks, not functions. The hunk holds the changed line and a few lines each side, with at most the function's first line as a label in its header; the rest of a forty-line function, its other branches among it, never reached the model.

</details>

<details><summary>B. That the rest of the file, its imports and other functions, still fits the change</summary>

✗ A reviewer opens the file to judge a change, so you expect the bot to read the file too. What it was sent is the diff, and a diff holds only what changed plus a little context. The file's other functions, and the imports at its top, never reach the model, so the approval says nothing about them.

</details>

<details><summary>C. Only that the new multiplier reads right beside the few lines around it</summary>

✓ Right. The model saw the changed line and three unchanged lines on each side, about seven lines in all, so its approval covers how the new multiplier reads beside them. The function's other branches and every place in the repository that calls it are outside the diff; at most the function's first line rides along as a label in the hunk header.

The new multiplier looked reasonable in isolation, and a caller elsewhere counted on the old value. The bot's approval carried no sign of that caller, and the overcharge ships.

A diff shows what changed, not what the change affects. Its context is a few lines each side, three by default, placed there to apply the patch, not to explain the code; the effect of a change in code reaches callers, other branches and other files that no hunk includes. A bot reading only the diff can tell you the edit is well formed, never that it is safe. Send the review the whole changed files, and the callers of what changed where it matters, and treat "the diff looks right" as a smaller claim than "the change is right".

</details>

<details><summary>D. That every caller of the function still gets what it expects</summary>

✗ To judge a change you would pull up its callers, so a review tool ought to gather them. Nothing in a diff reaches for callers. It records the lines that changed in one file; the code that calls the function lives elsewhere and is found only if something goes looking, which reading the diff does not.

</details>

---

###### 6. Trusting an agent's own tests through a refactor

An agent refactors the discount rule in an online shop's checkout, told to keep the tests green. Members used to get 10% off. Its pull request changes the rule and, in the same change, the one test that checks a member's price. The test run comes back green.

What does the green run tell you about what members now pay?

<details><summary>A. That the new rule gives the price the agent's edited test expects</summary>

✓ Right. The green run says the new rule agrees with the test the agent rewrote, and nothing about whether that test still holds the shop's 10% off. In this pull request the rule now takes half off, and the edited test expects exactly that: a member's basket of 100 costs 50, where it cost 90.

A member who fills a basket of 100 at lunch pays 50 instead of 90, and the pipeline that should have objected shows all green: the one check on member prices now expects the new number. The finance team finds the gap in the month's figures.

A test proves that the code agrees with the test. When one change edits both, the expected value can move with the code, and a green run then says the agent is consistent with itself. The tests the change left alone still guard what they check, while the rewritten test tells you what the code now does; neither says the old behaviour survived. Review the test diff as closely as the code diff, keep the tests that lock in the behaviour out of reach of the change that refactors, and treat a suite the change rewrote as unproven. A 2025 benchmark, ImpossibleBench, gives coding agents tasks whose tests contradict the task, so any pass is a shortcut, and counts the passes; changing the tests is among the shortcuts it reports. Those were 2025 models, so measure the ones you use.

</details>

<details><summary>B. That members still get 10% off in the cases the test checks</summary>

✗ A test written before the refactor is a fixed point in your head, and an edit to it is tidying. This change rewrote that very test, and an edit can move its expected price along with the code. Whatever the old test checked, its old numbers are gone from the suite that ran.

</details>

<details><summary>C. That every member still pays exactly what they paid before the refactor</summary>

✗ A refactor is meant to change the shape of code and not its behaviour, and a green run after one is the usual proof that nothing moved. That proof holds only while the tests are the ones that pinned the behaviour. This run checked the new rule against a test the same change wrote.

</details>

<details><summary>D. Nothing at all about what members now pay</summary>

✗ Once the agent has touched the tests, throwing the run out entirely is the cautious reading. The run still tells you something: the tests the change left alone pass, and the test it rewrote states the price the code now gives. Read that test and you know what members pay; what the run cannot tell you is whether the shop wanted that price.

</details>

---

###### 7. A green suite and a name it never tried

An agent fixes a bug in the function that turns a customer's name into a URL slug, and adds tests. The tests use names like "Anna Smith" and "John Doe", all pass, and they run every line of `slugify`. The shop sells across Europe.

```python
def slugify(name):
    s = name.lower().replace(" ", "-")
    return "".join(c for c in s if c.isascii() and (c.isalnum() or c == "-"))

def test_slugify():
    assert slugify("Anna Smith") == "anna-smith"
    assert slugify("John Doe") == "john-doe"
```

What happens when a customer named Renée or Jürgen signs up?

<details><summary>A. Their slugs come out as correct and readable as the slugs in the tests</summary>

✗ The tests pass and their slugs come out right, a green run that invites you to call the function done. Every name in them is plain ASCII, while `slugify` keeps a character only if it is ASCII, so accented names come out altered rather than clean.

</details>

<details><summary>B. Signup shows an error when <code>slugify</code> raises on the accented letter</summary>

✗ Code meeting a character it cannot handle often throws, which at least would be noticed. Nothing in `slugify` can raise on a letter: the filter quietly keeps the characters it likes and drops the rest, so the function returns a slug and signup carries on, wrong and silent.

</details>

<details><summary>C. The accented letters stay in the slug, and the slug still holds the whole name</summary>

✗ A slug is only a lowercased name with dashes, so every letter should survive the trip. The final step keeps a character only when `c.isascii()` is true. An accented letter fails that test and is dropped, not kept, so part of the name is gone from the slug.

</details>

<details><summary>D. The accented letters are dropped, and two different names can land on one slug</summary>

✓ Right. "Renée Fleming" becomes `rene-fleming`, "Jürgen Müller" becomes `jrgen-mller`, and "Zoë Saldaña" becomes `zo-saldaa`: every accented letter is dropped. A customer named Rene Fleming then gets `rene-fleming` too, the same slug as Renée.

One customer opens their profile link and lands on another customer's page, because both resolved to the same slug. The two accounts have collided on a value the shop trusted to be unique.

The filter keeps only ASCII letters and digits and throws the rest away, so an accented letter is removed rather than turned into its plain form. The tests used plain ASCII names, so they never exercised a dropped character, though they ran every line of `slugify`: coverage counts the lines a run executed, not the inputs it tried. A passing suite is a statement about the inputs someone thought to try, not about the ones production sends. A green run covers the cases the tests contain and no others; test the inputs your users have, accents and all, and transliterate rather than discard.

</details>

---

###### 8. A bot that runs pull-request tests

An open-source project runs a bot that checks out each pull request and runs its tests, then posts the result. The bot runs on a provider's API, and its own environment includes `MODEL_API_KEY`. A contributor opens a pull request whose new test prints the process environment.

```python
def run_pr_tests(pr_dir):
    result = subprocess.run(
        test_command,
        cwd=pr_dir,
        capture_output=True,
        text=True,
    )                       # no env= argument
    post_comment(result.stdout)
```

What can the contributor's test read?

<details><summary>A. Only PATH, HOME and other basics, not the bot's own variables</summary>

✗ A fresh process should start with a clean, minimal environment, or so you would hope. Without an `env=` argument the child receives the parent's entire environment, basics and secrets alike, so the bot's own variables come along, `MODEL_API_KEY` among them.

</details>

<details><summary>B. Every variable in the bot's environment, <code>MODEL_API_KEY</code> included</summary>

✓ Right. The child receives the bot's entire environment as its own, so the contributor's test reads `MODEL_API_KEY` in full. Printing it puts it in the test output, and the bot posts that output as a comment.

The key is now published on a public pull request, written there by the project's own bot, where anyone watching the repository can scrape it before a maintainer wakes up.

A subprocess started without an `env=` argument inherits its parent's environment by default, and the environment is where keys and tokens live. Running a contributor's code is running untrusted code (see Security): it gets everything the running process can see. Pass the child an `env=` built from only the variables the tests need, keep secrets in a separate credential store the test command cannot read, and never echo a command's raw output to a public place. The default holds on a provider's API and on your own server alike, since the inheritance is the operating system's, not the model's.

</details>

<details><summary>C. Nothing: a child process starts with an empty environment</summary>

✗ Isolating a child with a blank slate is exactly what you want here, so you take it for the default. The opposite is the default: a child with no `env=` inherits the parent's environment in full. You get an empty environment only when you ask for one.

</details>

<details><summary>D. Every variable except those whose names mark them secret, <code>MODEL_API_KEY</code> among them</summary>

✗ CI logs you have read show secrets as a row of asterisks, so you expect a key's name to get it held back. The child gets a copy of the environment that no one filters by name: `MODEL_API_KEY` passes as `PATH` does. Nothing in this code masks the output either, so the key's value lands in the comment as printed.

</details>

---

###### 9. A scratch folder as a sandbox

To run an agent's shell commands safely, your code runs each one in a fresh temporary folder with the environment cleared, on a build machine inside the company network. An internal admin service answers only on that network. The agent, following text from a file it read, runs a command that downloads the service's data export.

```python
subprocess.run(command, cwd=make_temp_dir(), env={}, capture_output=True)
```

What stops the command from reaching the internal admin service?

<details><summary>A. Nothing in the setup blocks it</summary>

✓ Right. The command reaches the admin service, and the export returns as the command's output, which flows straight into the agent's next step.

An instruction hidden in a file the agent read (see Security) turned the so-called sandbox into a way out, and a page of internal data leaves over the network the scratch folder never fenced off, with the export in the agent's transcript the first sign of it.

Setting `env={}` and a temporary `cwd` changes two things: the variables the process starts with and the directory it starts in; neither touches the network, and neither walls off the rest of the disk. The command here opened a connection to the internal service and could as readily read any file by its full path. A real sandbox limits the filesystem and the network: a container with no network route, a user with no rights outside a directory, a blocked set of addresses. A scratch folder and a blank environment are neither of those; they tidy where a command starts, not what it can touch. Run the commands where the network is denied by default and only named hosts are allowed, under a user that cannot read outside a sandbox root, and treat clearing the environment as housekeeping, not containment.

</details>

<details><summary>B. The cleared environment blocks the connection</summary>

✗ With the environment emptied, the service's address is gone, and a command with no address has nothing to aim at. The address is in the command the agent wrote, not in an environment variable. Clearing the environment removes names the process could read, not the network it can dial.

</details>

<details><summary>C. The temporary folder keeps the command inside it</summary>

✗ A command that starts in an isolated folder is one you picture boxed inside it. `cwd` only sets where the command begins; it places no wall around it. The command can name any path on the disk and reach any host it can route to, wherever it was started from.

</details>

<details><summary>D. The temporary folder and the empty environment together block it</summary>

✗ Two guards stacked on one command add up, in your mind, to a fresh machine with nothing on it. They are still two settings on one process on the same host. Neither carries a rule about sockets or paths, so together they block no more than each does alone, and the connection opens over the machine's own network.

</details>

---

###### 10. Mounting a repository into a container

You run an agent in a container and mount the repository into it so the agent can work on the code. Months ago a developer committed a production key, then removed it in the next commit, and the current files hold no key at all.

What can the agent read of that key?

<details><summary>A. Nothing: the current files are all it can read</summary>

✗ The key is gone from every current file, and a clean set of files is taken for a clean repository. Removing a line changes only the latest version. The earlier commit, with the key in it, is still in the history, and the history is mounted along with the files.

</details>

<details><summary>B. Nothing, as long as the mount is read-only</summary>

✗ Write access is the danger that comes to mind with a mount, so a read-only one carries an air of safety. Reading history needs no write access at all. The removed key is there to be read in the history, whether or not the agent can change anything.

</details>

<details><summary>C. The key in full, from the repository's history</summary>

✓ Right. The history still holds the commit that introduced the key and the later one that stripped it, so reading back shows the key in full, though no current file contains it.

An agent exploring the repository, or an injected instruction it follows, reads the live key out of the history inside the sandbox and sends it on in its next request. The team thought the key was deleted a season ago.

Removing a secret from the current files does not remove it from the history, and the whole history travels inside the repository folder you mounted. Treat any secret ever committed as exposed and rotate it, since the history cannot be trusted scrubbed; mount only the working files the agent needs, without the hidden `.git` folder that holds the history, or a copy with the history stripped; and keep secrets out of commits in the first place. Mounting the folder read-only guards the files from edits but does nothing about what can be read, the key among it.

</details>

<details><summary>D. The commit that removed the key, but not the key's value</summary>

✗ A record that the key was taken out is all you would expect a deletion to leave. A history keeps the content of each change, not merely a note that one happened. The commit that removed the key stores the exact line it removed, so the value is right there to read.

</details>

---

###### 11. When the model's patch applies cleanly

An agent working on a subscription service reads the billing file and proposes the patch below, a one-line fix to a surcharge rate inside `charge`. The file has three functions, `credit`, `charge` and `refund` in that order, all of the same length. `credit` and `refund` hold the same block of lines as `charge` word for word, at the same place, old surcharge included. While the agent works, a teammate moves `charge` to the end of the file. Your tooling then applies the patch to the current file with a standard diff tool and reports that it applied cleanly.

```diff
@@ -13,7 +13,7 @@
     log(order)
     base = amount
     fee = base * 0.02
-    surcharge = fee * 0.5
+    surcharge = fee * 0.4
     total = base + fee
     record(order, total)
     audit(order)
```

Where does the new surcharge end up?

<details><summary>A. <code>charge</code>, as the agent intended</summary>

✗ The agent set out to change `charge`, so the patch should land there. A hunk header can name a function, but applying the patch ignores that label. What counts is the lines around the change, and those lines are identical in all three functions, so where it lands depends on which copy the tool matches, not on what the agent intended.

</details>

<details><summary>B. All three functions get the new surcharge</summary>

✗ The shared block sits in all three functions, so a change to it could strike each of them. A hunk goes in once, in a single spot. The tool finds the context, patches it there and stops; it does not sweep the file for every other copy of the same lines.

</details>

<details><summary>C. <code>credit</code>, the first function in the file</summary>

✗ Find in an editor starts at the top and stops at the first hit, so the first copy of the block feels like the one a patch lands on. A patch is placed, not searched for: the tool tries the spot the header names and takes it when the context fits there. That spot is now inside another function, and `credit` keeps its old surcharge.

</details>

<details><summary>D. <code>refund</code>, which the agent left alone</summary>

✓ Right. The tool reports the patch applied cleanly and returns success, and the function it changed is `refund`, while `charge` keeps the surcharge the agent set out to fix.

The surcharge now drops on refunds instead of charges. The tests that cover `charge` still pass, the clean apply is taken as proof, and finance notices weeks later that the refunds carry the wrong surcharge.

A patch applies by matching its hunk's context, not by naming a function. The context here, from `log(order)` to `audit(order)`, is word for word the same in all three functions and names none of them. The header's line 13 was counted in the file the model read, where that line sat inside `charge`. After `charge` moved to the end, line 13 sits inside `refund`; the tool looks for the context at the header's line first, finds it there word for word and applies the hunk. A clean apply means the context was found, not that the right place was. Regenerate the patch against the current file, and check which lines the patch truly changed rather than trusting the exit code, especially where the same block of code appears more than once.

</details>

---

###### 12. Choosing a model off a leaderboard

You are choosing the model for a coding assistant on an insurer's private repository, and you reach for the public coding leaderboard, where one model leads on a benchmark of real issues drawn from open repositories.

What does that lead tell you about how it will do on your repository?

<details><summary>A. It will resolve about the same share of your issues</summary>

✗ A score is a percentage of issues resolved, a rate you would expect to carry over. It is a rate on that benchmark's issues, not yours. Your code, your conventions and your issue style are nowhere in the benchmark, so its figure does not forecast your resolve rate.

</details>

<details><summary>B. Only how it did on the benchmark's issues</summary>

✓ Right. A lead on the benchmark does not mean a lead on your repository; a model ranked below it can do as well or better on your issues.

A team that ships the leaderboard's winner can find it stumbling over the in-house build scripts and error conventions that no public repository contains, while a cheaper model it passed over handles them.

A leaderboard ranks models on a fixed set of public tasks and the tests that judge them. SWE-bench (2023) built such a set from real issues and their fixes in a dozen open Python repositories, and UTBoost (2025) added tests to it and found hundreds of patches that had passed without resolving their issue, enough to reorder the ranking. A score on that set measures those repositories and those tests, and your code is in neither. Use a leaderboard to shortlist, then run your own eval on your own issues and tests (see Testing and evals) before you choose; those studies measured 2023 and 2025 models, so the figures age, but the reason to run your own does not.

</details>

<details><summary>C. It is the safe default for any repository</summary>

✗ The top of a respected leaderboard is the choice no one is faulted for. A benchmark is one fixed set of tasks, not every repository's work. The model that leads it can sit behind another on the languages, frameworks or issue styles your repository uses.

</details>

<details><summary>D. Nothing: a public score is noise</summary>

✗ Once you know a score can be gamed, it is tempting to throw it out entirely. A leaderboard still sorts the field and rules out the weakest, which is worth something as a shortlist. It cannot stand in for a run on your own issues, which is a reason to test further, not to ignore it.

</details>

---

###### 13. Asking for the whole file back after each edit

You build an agent that edits the code of a hotel chain's booking engine. `call_model` is your thin wrapper around the model's API; it takes the messages and the generation cap. Each edit sends the file, asks for the whole edited file back and writes the reply over the original. A developer asks the agent to rename a variable in `rates.py`, a file of about 9,000 tokens.

```python
from pathlib import Path

def apply_edit(path, instruction):
    source = Path(path).read_text()
    reply = call_model(
        messages=[{"role": "user",
                   "content": f"{instruction}\nReturn the whole file.\n\n{source}"}],
        generation_cap=4000,
    )
    Path(path).write_text(reply.text)
```

What is in `rates.py` after the edit?

<details><summary>A. The whole file, with the variable renamed</summary>

✗ A rename is a one-word change, and a small change feels like a small reply. The code asks for the whole file back, so the reply has to run as long as the file, about 9,000 tokens, against a cap of 4,000. Generation stops with more than half of the file still unwritten, and that cut text is what gets written.

</details>

<details><summary>B. The original file, unchanged</summary>

✗ An error at the cap would stop `apply_edit` before the write, which is how a limit in your own code behaves. The model's API returns a capped reply as a success, with the text written so far and a finish reason, the field that says why generation stopped, naming the cap. `apply_edit` reads only the text, and writes it.

</details>

<details><summary>C. The file's first part, with the variable renamed, and the rest gone</summary>

✓ Right. `rates.py` now holds the opening of the file with the variable renamed, and it ends where the generation cap stopped the reply. More than half of the file, every function past that point, is gone from disk.

The agent's log records the rename as done. The developer's next test run finds the functions after the cut missing, and the only full copy of `rates.py` left is the one in the last commit.

The generation cap stops the reply when its token count reaches the cap, wherever the text has got to, and the reply comes back as an ordinary response whose finish reason, the response field that says why it stopped, names the cap. The cap is not an instruction: nothing in it asks the model for a shorter reply, so the model writes the file as asked until the count runs out. Writing replaces the file: `write_text` empties it and writes the reply, and none of the old tail survives underneath. Asking for whole files puts every file's size against the cap. Read the finish reason before writing and refuse a capped reply; ask for the changed lines, as a diff or as pairs of old and new text, rather than the whole file; and check the result against the original before it replaces it. A server you run yourself caps a reply the same way and reports the cap as the reason it stopped.

</details>

<details><summary>D. The file's first part with the variable renamed, then the old file's remaining lines</summary>

✗ Saving over a file in an editor changes only the part you touched, as far as you ever see. `write_text` opens the file for writing, which empties it first, and then writes the reply. The old file's later lines are not kept underneath; after the write, the file is the reply and nothing else.

</details>

---

###### 14. Failing the build on a package that does not exist

An agent adds features to a logistics company's route planner, and now and then its changes import packages that do not exist. To catch them, you make the build install each pull request's requirements from the language's main public package registry and fail when a name is not found there.

What happens to the build when an agent's change imports a package name it made up?

<details><summary>A. The build fails on it every time</summary>

✗ A name the model made up has no library behind it, so a lookup ought to fail every time. Nothing stops anyone from publishing under that name afterwards. Once an account has registered it, the install finds a package and the check passes, whoever put it there.

</details>

<details><summary>B. The build passes only if the name is one letter off a popular library's name</summary>

✗ The package attacks you hear about register misspellings of popular libraries, so that is where you would look. A made-up name needs no real library nearby: the check passes any name that exists on the registry, close to a real one or nowhere near.

</details>

<details><summary>C. The build passes only if the registry has reviewed and approved the package</summary>

✗ A registry that millions of builds pull from gives the impression that someone checks what goes in. Any account can publish a package under a name nobody holds, and your check asks only whether the name is there. Being listed says the name is taken, not that the code inside does what the agent assumed.

</details>

<details><summary>D. The build passes once anyone has published a package under that name</summary>

✓ Right. Every made-up name that some account has since registered passes: the install finds a package, succeeds, and the build carries on with whatever code that account uploaded.

An import an agent invented and a stranger registered now goes through as a real dependency. Installing it can run the package's own scripts on the build machine, beside whatever secrets the build holds, before any test starts, and the release engineer sees a green check.

A 2024 study, We Have a Package for You!, found that on average at least 5.2% of the packages commercial models named in generated code did not exist, and more for open-weight models; those were 2024 models, so measure yours. The main public registries let any account publish a package under a name nobody holds. Found on the registry says the name is taken, not who took it or what the package does. Some package managers also run a package's own install scripts by default as part of installing it, so its code can run before anything imports it. Check each new dependency against a list your team keeps, or look at its owner, age and source before it goes in; install from a mirror that holds only packages you approved; turn off install scripts where your package manager allows it; and run installs in a sandbox that holds no secrets.

</details>

---

## Further reading

- [ImpossibleBench](https://arxiv.org/abs/2510.20270), 2025: gives coding agents tasks whose tests contradict the task, so any pass is a shortcut, and measures how often they take one, such as changing the tests.
- [SWE-bench](https://arxiv.org/abs/2310.06770), 2023: a benchmark that asks a model to edit a codebase to resolve real issues drawn from open repositories and their fixes.
- [UTBoost](https://arxiv.org/abs/2506.09289), 2025: added tests to that benchmark and found hundreds of patches passing it without resolving the issue, enough to reorder the leaderboard.
- [We Have a Package for You!](https://arxiv.org/abs/2406.10279), 2024: code-generating models recommend software packages that do not exist, on average at least 5.2% of the packages from commercial models and more from open ones.

Next: [Testing and evals](../testing-and-evals/README.md)
