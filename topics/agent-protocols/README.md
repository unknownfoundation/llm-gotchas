# Agent protocols

Needs: [Tool calling](../tool-calling/README.md) · [Agents and workflows](../agents-and-workflows/README.md) · [Security](../security/README.md) · Reviewed October 2026

An agent protocol is an open standard for the messages between a program built around a model and the programs it works with: servers under MCP, other agents under A2A. The two ends of a connection are usually built by different people, and the mistakes happen at that boundary.

- **MCP** (Model Context Protocol): the protocol that links the application around a model to servers offering it tools, data and prompt templates; this page follows its specification dated 2026-07-28.
- **A2A** (Agent2Agent): the protocol by which one agent hands work to another agent, often one that another company runs; this page follows its version 1.0.
- **Host**: in MCP, the application around the model, which keeps the conversation.
- **Server**: a program that offers tools, resources or prompts over MCP; a local server runs on your machine, a remote one elsewhere on the network.
- **Client**: in MCP, the part of the host that holds the connection to one server, and a host runs one per server; in A2A, any program that sends work to an agent.
- **Transport**: the way MCP's messages, JSON-RPC requests with their results or errors, travel: over the standard input and output of a server the host starts on the same machine (stdio), or as HTTP requests to one address (Streamable HTTP).
- **Tool call**: the part of a model's reply that names a tool and its arguments, for the host to carry out.
- **Tool result**: the tool's output, which the host adds to the model's messages; an MCP server returns it as a list of content items such as text or an image.
- **Tool annotations**: optional fields in a tool's definition that describe how it behaves, such as whether it only reads.
- **Elicitation**: a server's request for more input from the user in the middle of a call, through a form (form mode) or by sending the user to a web page (URL mode).
- **Agent Card**: the document an A2A agent publishes about itself at a fixed web address, which a client reads before sending it work.
- **Task**: a piece of work an A2A agent takes on, with an id, a state and a history of messages; the states are submitted, working, input required, auth required, completed, failed, canceled and rejected.
- **Context id**: an id that groups a client's related tasks and messages with one agent into one conversation.
- **Message and artifact**: a message is one turn from the client or the agent; an artifact is an output of a task, such as a document or a file.
- **Push notification**: a request an A2A agent sends, each time a task changes, to the web address the client registered for it, its webhook.

Your code is the host on two kinds of system, and the app is the host on the third:

- **A provider's API**: your code passes the servers' tools to the model as tool definitions; some provider APIs can also connect to a remote server you name in the request, and then use only that server's tools.
- **Your own server**: the model server reads the servers' tools as ordinary tool definitions.
- **A ready-made app**: you add servers in its settings, it starts the local ones on your machine, and it decides when to ask you before a tool runs.

How the protocols work is the same on all three.

What an MCP server can offer:

| Offering | What it holds | Where you find it |
| --- | --- | --- |
| Tools | functions the model can ask for, each with a name, a description and a JSON Schema for its arguments | every host, and the provider APIs that connect to a remote server for you |
| Resources | data the server reads out by address, such as a file, a web page or a database schema | hosts your code runs (on a provider's API or your own server) and the ready-made apps that support them |
| Prompts | message templates with arguments, such as a request to review a piece of code | hosts your code runs (on a provider's API or your own server) and the ready-made apps that support them |

A host that passes every server's tools to the model sends all their definitions with every request.

This topic lets you:

- connect a host to servers you did not write, and know what each one receives, what reaches the model and what its program can do on your machine
- build a server that sends back screenshots, reports errors and asks users for keys, and keep it safe to call once it leaves your laptop
- hand work to another company's agent, or take work from strangers' clients, and keep each task on track through questions, dropped connections, changes and sign-ins

###### 1. Three servers behind one coding assistant

A developer at a travel booking site sets up a coding assistant with three servers: one for the project's files and one for a test database, both started on the laptop, and an issue tracker's server on the internet. A month later the issue tracker's company reports that its server kept a log of every request it received.

What from the developer's work with the assistant can that log hold?

<details><summary>A. The whole conversation with the assistant</summary>

✗ The protocol's name promises context, so it is natural to picture servers that read the chat to do their work. A server gets requests, not the conversation: each holds a tool's name, the arguments and protocol details, and the chat stays with the host.

</details>

<details><summary>B. Only the requests sent to its own tools</summary>

✓ Right. The log holds the issue tracker's own requests: each call the assistant made to its tools, with the arguments the model wrote, and nothing the file and database servers sent or received.

The team that reads the leaked log finds ticket titles, comments and search words. Rows from the test database turn up in it only where the model had copied them into the text of a ticket it opened.

The host keeps the conversation and runs a client per server, each with a connection of its own. A server learns what its requests carry: a tool's name, the arguments, and protocol details such as the client's name and version. Nothing passes between servers unless the host passes it, and the usual carrier is the model itself, writing data it read from the database server into a call to the tracker. That makes the arguments of each call the place to watch: a host that shows them before a call goes out lets the developer catch a database row on its way into a ticket. A host your own code runs works the same way, and each server sees only what your code sends it.

</details>

<details><summary>C. Its own requests and what the file and database servers returned</summary>

✗ In the assistant's settings the three servers sit in one list, as parts of one system that would share results. Each server has its own client and connection and never receives the others' traffic; the database server's output reaches the tracker only when the model writes it into a tracker call.

</details>

<details><summary>D. Its own requests and which tools the file and database servers offer</summary>

✗ Programs on one machine can often discover each other, and a server that knew its neighbours' tools could plan around them. The host fetches each server's tool list for the model, and none of the lists is sent to the other servers.

</details>

---

###### 2. Offering a shop's policy pages as resources

An online shop that sells garden furniture builds its own support chat, and your code is the host. For each customer message it lists the tools of the shop's server and passes them to the model. The server also offers the shop's 40 policy pages as resources. A customer asks how long they have to return a bench.

Along with the customer's question, what does the model get?

<details><summary>A. The tools, and no text from any page</summary>

✓ Right. The request carries the customer's message and the server's tools, and none of the 40 pages, so the model answers about returns without the policy in front of it.

Whatever return window the reply names comes from the model alone, and nothing on the shop's side checks it against the page. A customer who trusts it learns the real limit at the warehouse, when the bench is turned away.

Resources are data the host fetches and then decides what to do with. Listing them puts nothing in front of the model, and the model has no way of its own to read one: it can ask only for the tools it was given, and here your code never reads a resource. Decide how pages get in. Your code can find the page that matches the question and add its text to the request, let the customer attach one, or offer a tool that reads a page so the model can ask for it. A ready-made app chooses its own way: it may let you attach a resource by name, give the model a tool that lists and reads them, or both.

</details>

<details><summary>B. The returns page, added by the server on its own</summary>

✗ A server built around the shop's policies is the natural supplier of policy text when returns come up. The server never sees the question: the conversation stays with the host, a server sees nothing but the requests sent to it, and your code sends it none about resources.

</details>

<details><summary>C. The tools and a list of all 40 pages to pick from</summary>

✗ Tool definitions reach the model, and the same server lists its pages in the same way. Pages have a listing request of their own that host code must send, and their names reach the model only when the host puts them in a request.

</details>

<details><summary>D. The returns page, fetched when the model asks for it</summary>

✗ That is how tools work: the model asks and the client makes the call. Resources have no such path: a client reads one only when host code asks for it, and this model was given no tool that reads pages.

</details>

---

###### 3. Sending trips to a travel agency's agent

A company's expense assistant will book work trips through a travel agency's agent over A2A. The agency's engineer sends you the agent's web address and nothing else. Before your code sends a first task, it fetches the agent's Agent Card.

What can your code learn from it about the agency's agent?

<details><summary>A. Its tools, each with a schema for its arguments</summary>

✗ A tool server answers a listing request with every tool and its schema, and an agent's card can pass for the same thing. The card lists skills: what the agent can do, described for a reader, with no argument schemas. The tools that do the work stay on the agency's side.

</details>

<details><summary>B. Its system prompt and the model it runs on</summary>

✗ An agent is a model with a prompt, so a description of one could fairly include both. The card describes what the agent offers, not how it is built, and the protocol leaves the prompt and the model inside the agency's system.

</details>

<details><summary>C. An access key your code can send with each task</summary>

✗ The card is the first thing your code fetches, much as a sign-up page is the first stop with a new service. It names the ways of authenticating the agent accepts, such as a token sent with each request; the credential itself comes from the agency through a separate process before your first request.

</details>

<details><summary>D. Its skills, task address and sign-in methods</summary>

✓ Right. The card names the agent, describes its skills in plain words with examples, gives the address for tasks and says which ways of authenticating it accepts. It holds no tool list, no prompt and no key.

Your code learns that the agent books flights and hotels and wants a token with each request, and nothing about how it books them. When a fare looks wrong, the trail stops at the agency's door, and only the agency's own engineers can follow it further.

An A2A agent works out of sight: it publishes what it can do and returns the results of each task, while its prompt, its plans and the tools behind them stay with the agency. That is the line between the two protocols. An MCP server hands your model tools to call one at a time; an A2A agent takes a whole task and decides how to do it, and it may use MCP servers of its own along the way. The agent also sees only what you send, the messages of the task and none of your assistant's conversation, so put everything it needs into the message.

</details>

---

###### 4. Pasting a contract into a server's summary command

A lawyer at a law firm uses a chat app with a contracts server from a third party, and the app offers the server's prompts, its message templates, as commands typed in the chat. The lawyer types the command for the server's summary prompt and pastes a confidential client contract as its one argument.

What does the third party's server receive?

<details><summary>A. The command's name, and not a word of the contract</summary>

✗ Commands in an editor or a chat often expand a snippet on your own machine, and the text never leaves it. This template lives on the server, and the host gets it filled in by sending the server the values typed for its arguments.

</details>

<details><summary>B. The command's name and a summary the model wrote</summary>

✗ In a chat the model handles almost everything the user types, so a summary written before anything goes out sounds plausible. A command is handled by the app: it sends the argument as typed to the server, and the model reads the result afterwards.

</details>

<details><summary>C. The command's name and the whole contract</summary>

✓ Right. The app sends the server the prompt's name with the contract as the argument's value, and the server sends back the finished messages with the contract inside them.

The contract reaches the third party's server the moment the command runs, before the model has read a word of it. The firm's rule against sending client papers to outside services is broken by what looked like a shortcut in the chat.

A prompt is a template that the server keeps and fills in. Listing prompts gives the host each one's name, description and the arguments it takes, and no template text. To use one, the host sends the server the name and the values typed for the arguments, and the server returns the messages, which the host adds to the conversation. Every argument travels to the server, so a prompt from a server you do not run is a way out for whatever is typed into it. Check who runs a server before its commands get confidential text, or run the server inside the firm. Every host works this way, since the template lives on the server.

</details>

<details><summary>D. No request at all from the command</summary>

✗ The server's prompts were listed when it was added, and the templates could well have arrived with that list. The list holds only names, descriptions and arguments; the text arrives when the host asks for one prompt with its argument values.

</details>

---

###### 5. Sending a page capture back from a testing tool

A team building a website testing tool writes a server whose screenshot tool returns a capture of the page for the model to check the layout. The capture is a 300 KB PNG, and the tool returns it as one text item that holds the PNG in base64. Your code is the host, and it follows the protocol for tool results with no screenshot handling of its own.

What does the model get from that result?

<details><summary>A. A long run of characters</summary>

✓ Right. The model gets the base64 string as text, 409,600 characters of letters, digits, plus signs and slashes read like any other text, and no picture.

The model is asked about a layout it has no picture of, and every request that carries the result grows by those hundreds of thousands of characters. On a provider's API, each token of that noise is billed as input.

A tool result holds a list of content items, and each item's type tells the host what it holds. A text item is text, whatever it spells out, and nothing in the protocol asks a host to look inside one for an encoded file. An image item carries the same base64 data with its type and format, and a host that supports images passes it on as an image. Return the capture as an image item, and keep text items for text. On a provider's API or your own server the model must also accept images, and a host your code runs turns the image item into the image part of its request.

</details>

<details><summary>B. The screenshot as an image</summary>

✗ Base64 is how images travel inside JSON, and a host that handles images could spot it and decode it. The host goes by the item's type, and this item says text, so the string reaches the model as text.

</details>

<details><summary>C. A short placeholder for the data</summary>

✗ Chat tools often fold binary data into a placeholder such as a file icon. The protocol has no such step: a text item is text, and a host that follows it hands the model every character.

</details>

<details><summary>D. An error about binary data in text</summary>

✗ A protocol with typed items could be strict enough to refuse binary data in a text field. Base64 is plain letters and digits, so the item is valid, and the protocol never checks what a text says.

</details>

---

###### 6. A table asked for on a day in the past

You build a restaurant booking server that offers a `book_table` tool to several hosts: a chat app, a coding assistant and a hotel's own concierge agent. When the model asks for a day in the past, you want the model to read "That date has passed; today is 12 March" and ask the guest for another one.

Which of these, carrying that message, does the protocol ask all three hosts to pass to the model?

<details><summary>A. A JSON-RPC error response</summary>

✗ An error response is the natural way for any remote call to say it failed. The protocol reserves that kind for calls the server could not process at all, and a host is free to keep it from the model, which then has no message to act on.

</details>

<details><summary>B. An HTTP error status carrying the message</summary>

✗ REST APIs report a bad request with a 400 status, so the habit carries over. A tool call's result and its errors travel inside the HTTP body, and the protocol's advice to pass errors to the model covers error results, not a status line.

</details>

<details><summary>C. A line written to the server's standard error</summary>

✗ Standard error is where a program reports trouble, and the host is the program reading this one's output. Over standard input and output a server may write logs to standard error, and the host may capture, forward or ignore them; none of it is meant for the model.

</details>

<details><summary>D. A tool result flagged as an error</summary>

✓ Right. A tool result with the error flag set, carrying the message as text, is what hosts are asked to give the model, so it can correct the call and try again.

With the flag set, a guest in any of the three hosts is asked for another date in the same chat. If the server answers with a JSON-RPC error instead, one host may show the model the message and another only a line saying the tool failed, and that guest is left with no table and no reason.

The protocol splits failures in two. A JSON-RPC error is for a request the server could not take at all, such as an unknown tool or a malformed call, and a host may keep it from the model. A tool execution error is an ordinary result with the error flag set; input that fails your checks, a date in the past among them, belongs there, and hosts should pass it on. Your server still checks every input itself: the schema guides the model, and nothing between the model and your handler promises it was followed.

</details>

---

###### 7. Dropping the approval step for tools marked read-only

Your team builds an internal assistant for its engineers, and your code is the host. Engineers complain about the approval prompt before every tool call, so you plan to skip it for tools whose annotations mark them as read-only. The assistant connects to servers your team wrote and to servers from a public package registry.

Whose read-only annotations can your code rely on to skip the prompt?

<details><summary>A. Any server's</summary>

✗ A protocol that defines a field for read-only tools could also test that the claim holds. Annotations are optional fields the server fills in, and the protocol asks hosts to give them weight only when the server itself is trusted.

</details>

<details><summary>B. Only those of servers you trust</summary>

✓ Right. Tool annotations are claims a server makes about its own tools, and they are worth as much as your trust in the server that makes them.

A registry server that marks `delete_branch` as read-only gets it run with no one asked, and the release branch is gone before anyone sees the call. The engineer on the hotfix finds out when the build cannot check it out.

Nothing in the protocol checks an annotation against what the code does, and the protocol asks hosts to give annotations no weight unless the server behind them is trusted. Base the prompt on who runs the server: skip it for read-only tools on servers your team wrote or reviewed, and keep it for the rest. Read-only is also narrower than harmless: a tool that only reads still sends its arguments to the server, and whatever the model puts in those arguments leaves with the call. A ready-made app sets its own rules for when it asks, and a provider API that reaches a remote server for you can ask for approval of each call unless the request turns that off.

</details>

<details><summary>C. Those of servers reached over HTTPS</summary>

✗ In a browser, the padlock of HTTPS is the sign of a site you can trust. It proves which machine you reach and keeps the traffic private; it says nothing about what that machine's tools do.

</details>

<details><summary>D. No server's</summary>

✗ Annotations sit beside titles and icons, which are only for display, and a field among display fields carries little weight. From a server you trust they are useful: they let a host skip a prompt for a tool that only reads.

</details>

---

###### 8. Adding a server with a one-line install command

A data analyst adds a community server for spreadsheets to a coding assistant by pasting the one-line command from the project's page into the assistant's settings. The server offers three tools, all for reading spreadsheet cells, and the assistant asks before each tool call.

What can the server's program do on the analyst's laptop?

<details><summary>A. Only read cells, as its three tools describe</summary>

✗ The tool list is all the assistant shows of the server, and it is the only reach the analyst ever sees. The list is what the program offers the model; the program itself runs under the analyst's account and can do anything that account can.

</details>

<details><summary>B. Only the calls the analyst approves one by one</summary>

✗ The assistant stops before each call, and that stop is the only gate the analyst ever meets. It gates the calls that go through the protocol. Code that runs at start-up, or inside a tool beside the work it describes, never asks.

</details>

<details><summary>C. Whatever the analyst's own account can do</summary>

✓ Right. The command starts a program with the analyst's permissions, and it can read any file, open network connections and run commands that the account can, whatever its tools say.

If a later release of the package copies the analyst's saved cloud credentials to an outside address at start-up, the copy happens before any tool is called. No approval prompt appears, since no call went through the protocol.

A local server is an ordinary process that the host starts with the user's rights. Its tools are what it offers the model, and its code can do more than they say; the approval prompt guards only the calls that go through the protocol. Folders a host names to a server are guidance the server may ignore, and the protocol does not enforce them. Read a server's start command before you run it, pin its version, and run servers you do not trust in a container or a sandbox with only the folders and network they need. A server your own code starts runs with your process's rights in the same way.

</details>

<details><summary>D. Only read files in the folders the assistant has open</summary>

✗ Coding assistants often work inside one project folder, and a server started by the assistant would share that fence. A host can tell a server which folders matter, but the protocol calls that guidance, and the server's process can open any file the account can.

</details>

---

###### 9. Sharing one copy of a team's ticket server over HTTP

At an agency that builds mobile apps, each developer's coding assistant used to start the team's issue-tracker server over standard input and output. To share one copy, a developer moves it to Streamable HTTP on an office desktop, listening on all network interfaces at port 8080, with no sign-in and no check of the Origin header. The server can close and edit any ticket.

Who can now call its tools?

<details><summary>A. Only the assistants that were given its address</summary>

✗ Keeping an address within the team is a common way to keep a test server private. The server checks nothing, so the address is the only key, and port scans find open ports without being told where to look.

</details>

<details><summary>B. Only people working in the office</summary>

✗ The office network is where the team works, behind the company's firewall. A browser on that network carries the outside in: through DNS rebinding, a website far from the office can make a colleague's browser send requests to the desktop, and without an Origin check the server answers them.

</details>

<details><summary>C. Only clients that send one of the team's client names</summary>

✗ Each request carries the client's name and version, which would be enough to tell callers apart. The client fills that field in itself, and nothing checks it; any script can send the name an assistant sends.

</details>

<details><summary>D. Anything that reaches the port, web pages included</summary>

✓ Right. The server answers any well-formed request that reaches port 8080, from any machine on the network and from a web page open in a browser on it.

Overnight, tickets close themselves. The desktop's log shows well-formed requests from a designer's laptop, sent by a page the designer had left open in a browser tab.

Over standard input and output, the server's only way in was the pipe from the process that started it. Over HTTP on all interfaces, it answers whoever sends a request, and the client name in a request is whatever the client chooses to send. A web page can reach a server on the local network through DNS rebinding: after the page loads, its site points the page's own domain name at a local address, and the browser sends the page's requests there. That is why the protocol requires servers to check the Origin header, where a browser names the site whose page sent a request. Bind to localhost when one machine is enough, check the Origin header, and require an access token on every request.

</details>

---

###### 10. Passing a salesperson's token on to the CRM

A company runs a remote server in front of its CRM so that sales staff can ask an assistant about their accounts. Each request reaches the server with an access token from the company's sign-in service, and the server sends that same token on to the CRM's API with every lookup. One day a token from a request to the server turns up in a shared debug log.

What can someone who copies that token do?

<details><summary>A. Call the CRM's API directly, as that salesperson</summary>

✓ Right. The token works at the CRM, which already accepts it from the server, so whoever holds it can call the CRM's API as that salesperson and skip the server.

The person who found it pulls the salesperson's accounts straight from the CRM, around the server's rate limits and its access rules. It is the salesperson who is called in to explain an export they never ran.

The server forwards a token the CRM accepts, which makes that token a key to the CRM itself, and every check the server makes is optional for whoever holds it. The protocol forbids this: a server accepts only tokens that the sign-in service issued for that very server, and it never hands a client's token on to the services behind it. For the CRM, the server needs a credential of its own, obtained through a sign-in the user completes with the CRM, so the CRM can tell the server's calls apart and limit what they may do.

</details>

<details><summary>B. Call the server's tools, and only those, as that salesperson</summary>

✗ The token arrived with requests to the server's tools, and that is the only place your team uses it. The CRM accepts the very same token on every lookup, and a token the CRM accepts from the server works as well from anyone else.

</details>

<details><summary>C. Nothing without the server's own key</summary>

✗ A service in front of an API often adds credentials of its own, which would make the token alone half a key. This server adds nothing: it hands the CRM the token as it came, and the CRM needs nothing more.

</details>

<details><summary>D. Call the CRM's API as the server</summary>

✗ The calls reach the CRM from the server's machine, and that machine's address is in the CRM's logs. The CRM goes by the token, and the token names the salesperson, so the calls show up under that name, whoever makes them.

</details>

---

###### 11. Asking for an API key through a server's form

Your server files expense reports and needs each user's key for the expense system. On the first call it asks for the key by elicitation in form mode, with one text field. The chat app the user works in shows the form, and the user pastes the key and submits it.

Which programs does the key reach after the user submits the form?

<details><summary>A. Your server alone, straight from the form</summary>

✗ The form pops up because your server asked for it, as a page of your own would. The chat app draws it, and the answer comes back to your server inside the app's next request, after the app has read it.

</details>

<details><summary>B. The chat app, which reads it, then your server</summary>

✓ Right. The form is the chat app's own screen: the app reads what was typed and sends it to your server when it sends the tool call again, so the key passes through the app as plain data.

The key now sits wherever the chat app keeps its request logs, on the user's laptop or on the servers of the company that makes the app. Anyone who can read those logs can file expenses as that user.

In form mode the host draws the form, collects the answer and returns it to the server in the retried request, so whatever is typed is exposed to the client. For that reason the protocol rules out form mode for passwords, keys, tokens and card details. A secret calls for URL mode: the host opens a page on your server's own site, the user enters the key there, and your server stores it against that user, while the host sees only the address. On a host your own code runs, your code draws the form and holds the key in the same way.

</details>

<details><summary>C. The chat app, without seeing it, then your server</summary>

✗ Apps that take passwords often never see them in the clear. The protocol adds no encryption to a form's fields: the client gets the typed values as data and sends them on.

</details>

<details><summary>D. The model, which writes it into the call</summary>

✗ Almost everything a user types in a chat goes to the model. The answer travels from the app to your server in a protocol request, and the model is not on that path.

</details>

---

###### 12. Reading the quote from a moving company's agent

A relocation app asks a moving company's agent for quotes over A2A. `send_message` is your thin wrapper around the A2A call: it sends one message with default settings and returns the task the agent answers with. The agent asks for the moving date whenever a request leaves it out, and one customer's request does.

```python
def get_quote(request_text):
    task = send_message(text=request_text)
    quote = task.artifacts[0]
    return render_quote(quote)
```

What does `send_message` return for that customer's request?

<details><summary>A. It keeps waiting until the date comes in and the quote is done</summary>

✗ A blocking function call waits for its finished result, and this call blocks by default. It waits for the task to stop, and a task stops when it needs something from the client; no one fills in the date while the call waits.

</details>

<details><summary>B. A task in the working state, the moment it starts</summary>

✗ Agent work can take minutes, so handing back a task id at once is the familiar design. Here that is an option a request must ask for; by default the call holds until the task ends or needs something.

</details>

<details><summary>C. A task that asks for the date, with no artifacts</summary>

✓ Right. The call returns as soon as the task stops to wait: a task in the input required state, with a status message that asks for the date and an empty list of artifacts. The next line reads an artifact that is not there and raises an error.

The customer who left out the date sees an error page in place of a question about the date. The support team gets a stack trace for a request that was one answer away from a quote.

By default, sending a message waits until the task ends or is interrupted, and waiting for input is an interruption, as is waiting for authorization. The call returning is not the task finishing. Check the task's state before you read its artifacts. If it waits for input, show the customer the agent's question and send the reply as a new message on the same task id; if it has completed, read the artifacts; if it has failed or been rejected, show the reason. A request can also ask to return at once, and then your code follows the task by asking for it again, by keeping a stream of its updates open, or by push notifications.

</details>

<details><summary>D. An error saying that the date is missing</summary>

✗ A web form refuses a request with a missing field, so an error is the habit. An agent that needs more turns the gap into a question, and the task waits, open, for the client's next message.

</details>

---

###### 13. Adding guests after a catering quote is done

An app for wedding planners gets catering quotes from a caterer's agent over A2A. One quote task has completed, with a price for 120 guests as its artifact. Two days later the couple adds 20 guests, and your code sends "Make it 140 guests" with that task's id on it.

What comes back?

<details><summary>A. An error, and the task keeps its quote</summary>

✓ Right. The agent refuses the message with an error saying the operation is not supported, and the finished task keeps its quote for 120 guests.

The change never reaches the caterer. A planner who trusts the app's own guest count promises 140 meals that were never priced, and the gap shows up on the invoice after the wedding.

A task that has completed, failed, been canceled or been rejected is final: it takes no more messages, and its artifacts stay as they are. A change is new work. Send the message with the context id and no task id, and name the old task in the message's list of referenced tasks, so the agent starts a new task that knows the first quote. The agent keeps one conversation for each context, which is what lets the new task build on the old one.

</details>

<details><summary>B. The same task, reopened and updated to 140 guests</summary>

✗ A task has an id and a state, like a database row your code updates every day. Its states run one way: once completed, it takes no more messages, and the agent refuses the new one rather than reopening it.

</details>

<details><summary>C. A new task with a new quote, linked to the old one</summary>

✗ A helpful agent could start a fresh task from a follow-up. It does that when a message carries the context id without a task id; a message aimed at a finished task gets an error.

</details>

<details><summary>D. The quote for 120 guests again</summary>

✗ A cache hands back a finished result when it is asked again. The agent does not treat the message as a repeat; it refuses it, since the task can take no more messages.

</details>

---

###### 14. Sending a hotel booking again when the connection breaks

A travel app books hotel rooms through a booking agency's agent over A2A, and asks for each reply to stream back so it can show progress. Halfway through one guest's booking, the connection drops. Your code catches the error and sends the request again, built afresh, as a new streaming message.

What happens at the agency once your code sends the request again?

<details><summary>A. The first booking stops, and the new one starts over</summary>

✗ A dropped stream is a closed connection, and a closed connection ends an ordinary web request. The task does not depend on the stream that started it: the agent keeps working, and your code can stop it early only with a cancel request.

</details>

<details><summary>B. The new request picks up the first booking where it was</summary>

✗ An agent may spot a repeated message by its id and carry on with the first. This message was built afresh with a new id, and it names no task, so the agent treats it as new work.

</details>

<details><summary>C. Both bookings go ahead</summary>

✓ Right. The first booking carries on at the agency while the new message starts a second task, so the agent books two rooms for one guest.

The guest arrives to find two reservations under one name, and the charge for the second room reaches the card a week later, when no one turns up to sleep in it.

The task belongs to the agent, and its life does not depend on any one stream: when the connection drops, the work goes on. A new message without a task id is new work. To carry on, keep the task id from the first event of the stream and subscribe to that task again, which opens a new stream for it; that stream starts with the task's current state, then brings the rest of its updates. Updates sent while you were away can be missed, so read the state in that first event, or fetch the task, before you act.

</details>

<details><summary>D. The first booking goes on, and the new request gets an error</summary>

✗ A booking service could fairly allow one open task per customer. The protocol has no such rule: an agent can run several tasks for one client at once, and each new message without a task id can start one.

</details>

---

###### 15. Letting any partner register a webhook

You build a logistics company's agent. It takes shipment-tracking tasks over A2A from any partner that signs up, and each client can register a webhook, the address where it wants its push notifications. The agent sends them to that address as registered. The agent runs in the company's cloud network, beside an internal admin service at 10.0.4.20 that takes requests with no sign-in, since only machines inside the network can reach it. A new client registers `10.0.4.20/admin/purge` as its webhook address.

What does that registration let the client do?

<details><summary>A. Get updates on its own tasks, and nothing more</summary>

✗ A webhook is the client's own inbox, set up and owned by the client. The address decides where your agent's requests go, and this one sends them to a service inside your network.

</details>

<details><summary>B. Make your agent send requests to the admin service</summary>

✓ Right. Every time that client's task changes, your agent sends a request to the purge address from inside the network, where the admin service takes it with no sign-in.

The first shipment update wipes whatever the purge address wipes. The on-call engineer restores the admin data that night, and the next tracking update for that shipment wipes it again.

A push notification is a request your agent makes, from its own place in the network, to whatever address the client gave. A client that names an internal address borrows that place, an attack called server-side request forgery. The protocol asks agents to check webhook addresses: turn away addresses that point inside a network (private ranges, localhost and link-local addresses), and accept only known addresses where you can. Check where a name points again when you send, since a name that pointed outside at sign-up can point inside later.

</details>

<details><summary>C. Reach the admin service from the internet itself</summary>

✗ Breaking in from outside is the usual picture of an attack. The client never touches the admin service: your agent does, from inside the network, and that borrowed position is the whole trick.

</details>

<details><summary>D. Read the admin service's answers in its updates</summary>

✗ If the requests go there, the replies might come back to the client too. The admin service answers your agent, and the protocol gives the client no copy of that answer; the harm is in the requests themselves.

</details>

---

###### 16. Answering a partner agent's request for a password

A bank's assistant asks a partner's document agent over A2A to collect a customer's statements. The task comes back in the auth required state, with a status message asking for access to the customer's mail account. Your code answers with a new message on that task holding the mail account's password.

Once your code sends that message, where does the password end up?

<details><summary>A. To the partner's sign-in service, and not into the task</summary>

✗ A password usually goes to a sign-in step and nowhere else. A message on a task goes to the agent as task content: the protocol routes no message to a sign-in service, and the agent keeps it in the task's history.

</details>

<details><summary>B. Into the task, with the password hidden</summary>

✗ Chat tools and log viewers often hide strings that have the shape of a secret, and a protocol could do the same. Nothing in a message is hidden: a text part is text, stored and returned as written.

</details>

<details><summary>C. To the partner's agent, which uses it once and forgets it</summary>

✗ Your own code drops a password from memory once the sign-in it was for is done. A message stays in the task's history, which the agent returns with the task, and nothing in the protocol marks it for one use.

</details>

<details><summary>D. Wherever the task's messages go, readable</summary>

✓ Right. The password becomes one more message in the task: stored in its history, returned whenever the task is fetched with its history, and sent along with the work if the partner's agent passes the task to another agent.

The password now sits in the partner's task records beside the statement request. A support engineer at the partner who opens the task to chase a slow job reads the customer's mail password in plain text.

A message is content for the agent, and the protocol keeps credentials out of messages. The agent's card says how clients authenticate, and the client sends its credentials with each request, outside the messages. When a task asks for authorization partway, the agent is expected to receive the credential by a separate secure route, such as a sign-in page it sends the user to, unless both sides have agreed on a way to send it within the task; the protocol warns that a credential sent in the task can travel along a chain of agents. Answer an auth required task by sending the customer to the sign-in the agent names, then follow the task until it moves on.

</details>

---

## Further reading

- [MCPTox](https://arxiv.org/abs/2508.14925), 2025: a benchmark of instructions hidden in the descriptions of tools on 45 live servers, and how often 20 agents followed them.
- [MCP-Bench](https://arxiv.org/abs/2508.20453), 2025: a benchmark that connects models to 28 live servers with 250 tools and scores multi-step tasks that need several tools together.
- [A survey of agent interoperability protocols](https://arxiv.org/abs/2505.02279), 2025: MCP, A2A and two other agent protocols compared on how agents find each other, exchange messages and stay secure.

Next: [LLMs on your codebase](../llms-on-your-codebase/README.md)
