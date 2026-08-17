# Video Walkthrough Script
### Inbound Lead Qualification & Routing Swarm

**Target runtime:** ~10 minutes
**Format:** screen recording of the n8n canvas, voiceover
**Audience:** someone who knows what n8n is but has not seen this workflow

---

## Before you hit record

- [ ] **Fund the OpenAI API key.** Section 7 shows a live run. Right now every run dies at the first agent call with `insufficient quota`. Without this, record everything except Section 7.
- [ ] Set `contactOwner` on **HubSpot — Assign to AE** (currently `REPLACE_WITH_AE_OWNER_ID`).
- [ ] Set `listId` on **HubSpot — Add to Nurture List** (currently `0`).
- [ ] Confirm the Slack channel is `sales-alerts`.
- [ ] Do one throwaway run first so you have a **successful execution** to open. Never demo a first run live.
- [ ] Zoom the canvas so all 14 nodes fit, then zoom into each node as you talk about it.
- [ ] Close any tab showing your API keys.

**Timing note:** section durations are guides. Sections 3 and 4 carry the value — if you run long, cut from Section 8, not from those.

---

## 1 · Cold open (0:00 – 0:25)

> **[SHOW]** Full canvas, zoomed out. Slowly pan left to right.

**SAY:**

"This is a lead qualification agent built in n8n. When someone fills in the contact form on our website, this workflow researches their company, scores them out of a hundred against our ideal customer profile, and then decides on its own whether a salesperson should hear about them today, or whether they go into a nurture sequence instead.

Fourteen nodes. Two stages. No human touches it until there's a draft email waiting for review.

Let me walk you through how it actually works, and why each piece is there."

---

## 2 · The problem (0:25 – 1:05)

> **[SHOW]** Stay on the full canvas.

**SAY:**

"The problem this solves is a boring one. Inbound leads arrive with almost no information — usually just a name, an email, and a company. Somebody on the sales team then has to go look up who these people are, decide whether they're worth a call, and most of that research turns out to be wasted because most inbound leads aren't a fit.

So the goal here isn't to replace the salesperson. It's to make sure that by the time a human looks at a lead, the research is already done and the obviously bad ones have been filtered out.

The workflow splits into two halves. The left half gathers information and turns it into a number. The right half acts on that number."

---

## 3 · Stage 1 — Enrich & Score (1:05 – 4:15)

### 3a · Lead Webhook

> **[SHOW]** Click into **Lead Webhook**. Show the HTTP method, the path, and the Respond setting.

**SAY:**

"Everything starts here. This is a webhook node listening for POST requests at `/webhook/inbound-lead`. You paste that URL into whatever builds your form — Webflow, Typeform, WordPress, it doesn't matter — and when somebody hits submit, the form posts the lead here.

Two details worth pointing out.

First, the incoming data lands under `$json.body`, not `$json` directly. That trips people up constantly — you look at the webhook output, you see the fields, you write `$json.name`, and it's silently empty. It's `$json.body.name`.

Second, look at the Respond setting: it's set to respond *immediately*. The workflow replies 'received' the instant the lead arrives, and then keeps working in the background. That matters because the AI research takes twenty or thirty seconds. If we waited for it, the person's browser would sit there spinning on the thank-you page. Nobody wants that."

### 3b · Enrichment Agent

> **[SHOW]** Click into **Enrichment Agent**. Open the system message so the prompt is visible. Then zoom out to show the two sub-nodes hanging off the bottom.

**SAY:**

"This is the first agent, and this is where the research happens.

An AI Agent node in n8n isn't a single call to a model — it's a loop. The model gets a goal and a set of tools, and it decides for itself how many times to use them before it's satisfied. Look underneath the node and you'll see two things plugged into it.

The first is the **model** — that's `OpenAI — Enrichment`, running gpt-5-mini. That's the brain doing the reasoning.

The second is the **tool** — `Search Google for company info`, which is SerpAPI. This is the agent's hands. Without it the model would just be guessing from training data, which for a small company is worthless and for any company is out of date. With it, the agent can actually go and look.

Now, the prompt. I'm telling it to run *several* searches, not one — the company's own site, headcount, funding, recent news, and the email domain to make sure we've got the right company and not a similarly-named one.

And then there's this section, which I'd argue is the single most important part of the whole workflow."

> **[SHOW]** Highlight the "Accuracy rules" block in the system message.

**SAY:**

"'Never invent a number. If you can't find headcount or revenue, write unknown explicitly.'

Here's why that matters. This agent only has web search. It does *not* have Clearbit or Apollo or any real firmographics provider. So for a lot of companies — especially small private ones — it simply will not find employee count or revenue. And a language model, asked for a number it doesn't have, will very happily produce a plausible-looking one.

If that happens, the next step scores a real lead against a completely fabricated company. So the agent is told to say 'unknown' and mean it. We'd rather have an honest gap than a confident lie.

The output of all this is a three-paragraph brief: what the company does, its firmographics, and any buying signals from the last two years."

### 3c · Score Lead

> **[SHOW]** Click into **Score Lead**. Scroll the system prompt to show the rubric. Then show the two sub-nodes.

**SAY:**

"Now we turn that brief into a number.

This is a Basic LLM Chain, not another agent — and that's deliberate. Agents are for when you need tools and multiple turns. This is one shot: read the brief, apply the rubric, output a score. No tools needed. Using an agent here would be more moving parts for no benefit.

The rubric is a hundred points across five dimensions. Company size is worth thirty, industry fit twenty-five, buying signals in the last two years another twenty-five, email quality ten, and geography ten. There are also hard disqualifiers — if the company can't be identified, or it's a job seeker, or a competitor, or a staffing agency, the score gets capped at twenty no matter how it scored elsewhere.

Now watch this part, because it connects directly back to the honesty rule from the last step."

> **[SHOW]** Highlight the "Unknown-data rule" block.

**SAY:**

"If the brief says a dimension is unknown, the rubric awards the *midpoint* of that dimension — not zero.

Think about why. If we scored unknowns as zero, then every company our research couldn't find would automatically look terrible. We wouldn't be measuring lead quality anymore, we'd be measuring how good our search was. A great prospect at a quiet private company would get buried. So an unknown costs the lead nothing — instead it lowers a separate `data_confidence` field, which tells you how much to trust the score.

Underneath the node, two sub-nodes again. The model, `OpenAI — Scoring`, running gpt-5 — a stronger model than the enrichment step, because this is the judgment call that everything downstream depends on.

And then the **Structured Output Parser**. This is what guarantees we get clean machine-readable JSON — a score, the reasoning, and the confidence level — instead of the model replying 'I'd rate this lead about an 85 out of 100!' in prose. You cannot branch on prose.

One more thing on the parser: it has auto-fix switched on. If the model returns slightly malformed JSON — a trailing comma, or wrapped in a markdown code block, which models love to do — the parser hands it back to a model to repair rather than crashing the entire workflow. Without that, one bad response takes down the run."

---

## 4 · Stage 2 — Route & Act (4:15 – 8:00)

### 4a · Normalize Lead + Score

> **[SHOW]** Click into **Normalize Lead + Score**. Show the seven assigned fields.

**SAY:**

"This node doesn't talk to any service. It's plumbing, and it earns its place.

At this point the data we need is scattered — the lead's name and email are back at the webhook, the research brief is in the agent's output, and the score is in the scoring chain. This node pulls all of it into one flat item: name, email, company, brief, score, reasoning, confidence.

The reason to do that is everything to its right. Five nodes downstream need this data. If each of them reached back separately into three different nodes, then the day something changes shape, I'd be fixing it in five places. This way there's one place to fix.

That's not hypothetical, by the way. The exact field the score arrives in depends on the n8n version, so this node handles both possibilities. One node absorbs that uncertainty instead of five."

### 4b · Hot Lead?

> **[SHOW]** Click into **Hot Lead?**. Show the condition. Zoom out to show the two output branches.

**SAY:**

"And here's the decision point — the whole workflow narrows to this.

It's an If node doing one numeric comparison: is the score greater than or equal to eighty? That's it. All that AI work upstream exists to produce a number that this node can compare.

Everything above the line takes the true branch. Everything below takes the false branch.

Eighty is a starting point, not a law. If your reps complain they're getting junk, raise it. If the nurture list is swallowing good leads, lower it. It's one number in one field — this is the single easiest thing in the workflow to tune, and it's the one you should expect to tune."

### 4c · The hot path

> **[SHOW]** Follow the top branch left to right, clicking each node in turn.

**SAY:**

"A hot lead triggers four things in sequence.

**First, HubSpot.** This creates or updates the contact and assigns it to an Account Executive. Note it's an upsert, not a plain update — if this person isn't in the CRM yet, which for inbound is likely, it creates them rather than failing. It also splits the full name into first and last, because that's how HubSpot wants it.

**Second, Slack.** A message into the sales channel with the name, company, score, confidence, the reasoning behind the score, and the full research brief. The point is that a rep can read that message on their phone and know whether to care, without opening anything.

**Third, the intro email gets written.** This is a second, much smaller LLM step. It gets the research brief and writes a first-touch email under a hundred and twenty words, opening on one specific real detail about the company.

It's under strict instructions: no flattery, no 'I hope this finds you well', and critically — it may not state a single statistic, headcount, or customer name that isn't in the brief. A hallucinated fact in an email that goes to a real prospect is genuinely embarrassing, and this is where that risk lives.

**Fourth, Gmail.** And this one is the most important design decision in Stage 2." 

> **[SHOW]** Zoom in on the Gmail node's operation field: **Create Draft**.

**SAY:**

"It creates a *draft*. It does not send.

The email lands in the rep's drafts folder, and a human reads it, edits it, and decides whether it goes out. The AI does the research and the typing; a person still makes the call. That's the line I'd encourage anyone to hold — the moment you let a model send cold email unsupervised, you're one bad generation away from a real problem with a real customer."

### 4d · The cold path

> **[SHOW]** Click the bottom branch — **HubSpot — Add to Nurture List**.

**SAY:**

"The cold path is deliberately boring. One node. It adds the contact to a HubSpot list that feeds a thirty-day marketing sequence.

No Slack message. No email draft. Nobody on the sales team is told this lead exists.

That's the entire point. The value of scoring isn't the hot leads — it's that the low-propensity ones never reach a rep's calendar. They still get nurtured, and if they warm up later they come back through the top of this same workflow."

---

## 5 · Reliability (8:00 – 8:45)

> **[SHOW]** Click a HubSpot node, open Settings, show **On Error** and **Retry On Fail**.

**SAY:**

"Two things stop this from being fragile.

Both AI steps and all three integrations retry twice before giving up. Model APIs and CRM APIs both have bad seconds, and a single blip shouldn't lose a lead.

More importantly — look at the error setting on HubSpot and Slack. It's 'continue'. These nodes run in a chain, so by default if HubSpot went down, the Slack alert behind it would never fire. You'd lose the notification for your best lead of the week because of an unrelated CRM outage. With continue-on-error, a HubSpot failure doesn't stop the alert from going out.

The way I think about it: the alert is the part a human depends on. It should be the hardest thing to break."

---

## 6 · The honest limitations (8:45 – 9:20)

> **[SHOW]** Back to full canvas.

**SAY:**

"Three things I'd want you to know before trusting this.

**One: the research is only as good as web search.** There's no Clearbit or Apollo here, so headcount and revenue are often genuinely unknown. The workflow handles that honestly rather than hiding it, but if firmographic precision matters to your scoring, adding a real data provider is the upgrade.

**Two: the rubric is an opinion.** Those weightings — thirty points for company size, twenty-five for industry — are a starting position, not truth. You should expect to argue with them after seeing fifty real leads.

**Three: watch the confidence field.** If you see a lot of low-confidence scores, the scores themselves aren't very meaningful yet. Fix the research before you trust the numbers."

---

## 7 · Live run (9:20 – 9:50)

> **[SHOW]** Open a **successful** past execution. Click through the nodes in order: webhook payload → the three-paragraph brief → the JSON score → which branch lit up → the Slack message → the Gmail draft.

**SAY:**

"Here's a real run. The lead comes in with three fields. The agent goes away and produces this brief — you can see it citing what it actually found. The scoring step turns that into this JSON: a score, the reasoning, and the confidence.

That number goes into the If node, this branch lights up, and here's the Slack message the rep got — and here's the draft sitting in the inbox waiting for them.

Total elapsed time, about thirty seconds. And nobody did any research."

---

## 8 · Close (9:50 – 10:05)

> **[SHOW]** Full canvas.

**SAY:**

"That's the whole thing. A webhook, two AI steps, one decision, and two different endings.

If you're building something like this, the parts I'd steal are the honesty rules in the prompts and the draft-instead-of-send at the end. The AI research is the easy bit. Making sure it doesn't confidently make things up, and making sure a human still signs off before anything reaches a customer — that's the part that decides whether you can actually put it in production.

Thanks for watching."

---

## Appendix · Node-to-purpose cheat sheet

Keep this open while recording.

| # | Node | Type | One-line purpose |
|---|---|---|---|
| 1 | Lead Webhook | `webhook` | Catches the form POST; data arrives at `$json.body`; replies 200 immediately |
| 2 | Enrichment Agent | `langchain.agent` | Research loop — decides what to search and when to stop |
| 3 | OpenAI — Enrichment | `lmChatOpenAi` | gpt-5-mini; the reasoning behind the research |
| 4 | Search Google for company info | `toolSerpApi` | SerpAPI; the agent's only source of real facts |
| 5 | Score Lead | `chainLlm` | One-shot grading against the ICP rubric |
| 6 | OpenAI — Scoring | `lmChatOpenAi` | gpt-5; shared by the parser and the email writer |
| 7 | Score Output Parser | `outputParserStructured` | Forces JSON `{score, reasoning, data_confidence}`; auto-fixes bad output |
| 8 | Normalize Lead + Score | `set` | Flattens everything into one item; single point of repair |
| 9 | Hot Lead? | `if` | The decision: `score >= 80` |
| 10 | HubSpot — Assign to AE | `hubspot` | Upserts the contact, assigns the AE |
| 11 | Slack — Alert Sales Team | `slack` | Posts the alert a rep can act on from their phone |
| 12 | Write Intro Email | `chainLlm` | Drafts <120 words; barred from citing facts not in the brief |
| 13 | Gmail — Draft Intro Email | `gmail` | **Draft only, never sends** — human signs off |
| 14 | HubSpot — Add to Nurture List | `hubspot` | Cold path; 30-day sequence, no rep involved |

### Phrases worth landing

- "The agent's hands, not just its brain." — introducing the SerpAPI tool
- "We'd rather have an honest gap than a confident lie." — the unknown rule
- "You cannot branch on prose." — why the output parser exists
- "One place to fix, not five." — the Normalize node
- "The AI does the typing. A person still makes the call." — Gmail draft
