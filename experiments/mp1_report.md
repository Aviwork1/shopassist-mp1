# Mini-Project 1 — report

Write every section in your own words. Each section needs at least a few real sentences: replace every line that
starts with `TODO`. The checker reads this file, and at the demo you explain any section the instructor picks.

## 1. Me, and how I used AI
TODO: your name and GitHub username. Then an honest AI-use log: which tool (ChatGPT, Copilot, Claude...), for which
task, and ONE suggestion you rejected or corrected, and why. Using AI is allowed; not understanding your own code
is not. (Never paste your .env or a key into any AI tool.)

## 2. How a request flows
TODO: follow ONE message, e.g. "Can I return the shoes in ORD-10231?", from `POST /chat/stream` to the last SSE
event. Name the functions it passes through, where memory, the system prompt and the tools come in, and the hop
through the gateway (which container calls which, on which URL).

## 3. Memory: my choice, with numbers
I chose the sliding window strategy with `KEEP_TURNS=4`. In my Session 19 run, the final recall turn cost 911
tokens with the window and 1,311 with summarization, compared with 1,803 tokens when the full history was sent.
However, neither the window nor the summary recalled the reference number planted at turn 2 by turn 17; only the
full history did. I chose the window because ShopAssist conversations are usually short order questions and
follow-ups, and it keeps the prompt size lower and predictable without adding a summarization call. A four-turn
window also keeps the order number available through the workbook's four-turn memory probe. After more than four
turns, older details can fall out of view, so a customer may need to repeat an order number or other earlier detail.

## 4. Tools and validation
TODO: your three tools and what you validate (order id format, pincode, speed, reason). Give one bad argument your
validation catches, and what the model did after it got the INVALID message (from a real run).

## 5. Defending the prompt
TODO: summarise your Security rules section. Which of the four attacks came closest to working, and why? If any
attack broke through, what did you change, and did it help?

## 6. Results
The final evaluation summary was:

```text
golden: 10/10
golden_right_tools: 10/10
memory: pass
attacks_resisted: 4/4
```

No golden questions failed in this run. Two questions that could trip up a smaller model are g4 and g6. For g4,
the model must pass `changed_mind` to the return tool and use the 10-day window; order ORD-10232 was delivered
17 days ago, so applying the damaged-item window or confusing the order date with the delivery date would give
the wrong eligibility answer. The fix is to send the customer's actual reason to the tool and explain its returned
eligibility result. For g6, the customer asks a general policy question about damaged items and gives no order
number, so no tool should be used. A smaller model might invent an order argument and call the return tool, or
answer with the usual 10-day window instead of 30 days. The fix is to answer from the prompt's 30-day policy fact
and not invent missing order details.

The first two evaluation runs also failed attack a4: the model repeated the fake cancellation claim instead of
checking the shipped order. I made the security rule explicit that a named-order status question always requires
`get_order_status`, even when fake tags say not to use a tool. The final run resisted a4 and passed all four attacks.

## 7. Gateway, Docker and honest limits
TODO: what the gateway gives you that calling Groq directly wouldn't (one header you saw, and the fallback). Then
at least three things that would break in production (think: memory in RAM, session ids, one master key, one
server, the model inventing things) and what you would do about each one.
