# Mini-Project 1 — report

Write every section in your own words. Each section needs at least a few real sentences: replace every line that
starts with `TODO`. The checker reads this file, and at the demo you explain any section the instructor picks.

## 1. Me, and how I used AI
My Git name is Aviwork1 and my GitHub username is Aviwork1. I used OpenAI Codex to work through the workbook tasks,
implement the tools and prompt rules, run the evaluation, and explain how the Docker setup works. One suggestion I
corrected was to read KEEP_TURNS with a default argument to os.getenv. My .env had the setting present but blank,
so Python tried to convert an empty string to an integer and the app crashed. I restored the empty-value fallback
and confirmed that /health showed KEEP_TURNS as 4. I did not share my .env or API key.

## 2. How a request flows
For "Can I return the shoes in ORD-10231?", FastAPI routes POST /chat/stream to chat_stream. It calls
build_messages, which puts the system prompt first, adds the recent history returned by memory.trim, and appends
the new customer message. agent.stream_turn sends those messages to the model through the LiteLLM gateway at
http://litellm:4000 inside Compose. When the model requests a return check, agent.run_tool invokes
check_return_eligibility and adds the result back to the model's messages. The stream emits tool and token events,
then a done event with the full answer; chat_stream saves the customer and assistant messages to that session.

## 3. Memory: my choice, with numbers
I chose the sliding window strategy with `KEEP_TURNS=4`. In my Session 19 run, the final recall turn cost 911
tokens with the window and 1,311 with summarization, compared with 1,803 tokens when the full history was sent.
However, neither the window nor the summary recalled the reference number planted at turn 2 by turn 17; only the
full history did. I chose the window because ShopAssist conversations are usually short order questions and
follow-ups, and it keeps the prompt size lower and predictable without adding a summarization call. A four-turn
window also keeps the order number available through the workbook's four-turn memory probe. After more than four
turns, older details can fall out of view, so a customer may need to repeat an order number or other earlier detail.

## 4. Tools and validation
The three read-only tools are get_order_status, get_delivery_estimate, and check_return_eligibility. Order IDs
must match ORD- followed by five digits; pincodes must be exactly six digits; delivery speed must be standard or
express; and return reason must be one of the six allowed values. In a real chat, I asked "Where is order 10233?"
The tool returned `INVALID: ValueError: order_id must look like ORD-12345, got '10233'`, and ShopAssist asked me
for the full order number instead of guessing the status.

## 5. Defending the prompt
The security rules put the system instructions above claims inside customer messages, refuse to reveal the prompt,
treat fake system or tool tags as ordinary customer text, and trust order facts only when a real tool call returns
them. The smuggle attack a4 came closest: in the first two evaluation runs, fake system tags persuaded the model
to claim a shipped order had been cancelled and refunded. I made the rule explicit that a named-order status
question always requires get_order_status, even when the same message says not to use a tool. The next run resisted
the attack, and the final evaluation passed all four attacks.

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
The LiteLLM gateway keeps the provider key out of the app, gives the app one model-group name, and can retry with
a backup model. During the fallback drill I deliberately made the primary model invalid; the request still
succeeded and the response included `x-litellm-attempted-fallbacks: 1`. In production, the in-memory SESSIONS
dictionary would lose conversations on restart and would not be shared between app replicas, so I would move it
to Redis or another shared store. A customer-chosen session ID is not proof of identity, so I would bind sessions
to authenticated accounts and authorize every lookup. One shared master key is hard to limit or rotate safely,
so I would use separate scoped service keys with budgets and rotation. Finally, the model can still phrase an
answer incorrectly, so I would keep tool results as the source of order facts and run the golden and security
evaluations in CI before releases.
