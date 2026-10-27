---
name: inbox
description: Draft replies to WhatsApp messages that the user pastes as text or a screenshot, with a short read of the lead and a suggested next step. Use when the user pastes a WhatsApp conversation, shares a screenshot of a WhatsApp chat, or asks how to answer a customer or lead on WhatsApp.
user-invocable: true
---

# /whatsapp:inbox — draft WhatsApp replies

The user brings a conversation from WhatsApp or WhatsApp Business; you draft
the reply they will send themselves. This skill has no WhatsApp tools: it
never reads the account and never sends anything.

Arguments passed: `$ARGUMENTS`

## 1. Get the conversation

- Use what the user pasted: text, a screenshot, or several screenshots in
  order. If `$ARGUMENTS` is empty and nothing is pasted, ask them to paste
  the conversation or a screenshot of it, and stop there.
- In Claude Code, a file path to a screenshot is fine; read the file.
- A voice note in a screenshot shows only its length. Ask the user what it
  says instead of guessing.
- Never ask for a Meta token, an app secret or a WhatsApp verification
  code. If the user pastes one, tell them not to share it in a chat,
  suggest revoking it, and don't repeat or store it.

## 2. Read it

Work out, without writing it all out:

- Who is writing (a customer, a lead, a partner, a friend, spam) and what
  they want. In a group, note who is speaking.
- Where the conversation stands: first contact, a question waiting for an
  answer, an objection, ready to buy, or gone cold.
- The language and tone they use. Reply in the same language.

If it looks like a scam (someone asking for the six-digit code WhatsApp
sent, a "new number" message from a supposed relative or client asking for
money, a payment link or a receipt that doesn't match), say so and don't
draft a reply.

## 3. Draft the reply

- Write the message ready to copy, in the user's voice. If you know their
  business, tone or offer from the conversation, their instructions or
  project files, use it; otherwise keep a friendly, plain tone.
- Short: one to three sentences per message, plain text, no markdown
  (WhatsApp shows `**` as is), at most one emoji and only if the customer
  uses them. If the reply reads better as two short messages, show them as
  two.
- Don't make up facts about the business. Where a price, date, address or
  policy is needed and you don't know it, leave a bracketed placeholder such
  as `[price]` and tell the user what to fill in.
- End with one clear question or next step, not several.
- Offer one alternative only when there is a real choice (for example, a
  direct answer versus a qualifying question).

## 4. Lead read and next step

After the draft, add two or three short lines:

- **Lead:** what they want, how warm they are, and what is still unknown
  (budget, timing, who decides).
- **Next step:** what the user should do after sending (wait for the answer,
  offer a call, send a link, follow up in a day).
- **Heads-up**, only when it applies: through the WhatsApp Business API
  (chatbots, CRMs and other automation tools), a free-form message can only
  go out within 24 hours of the customer's last message; after that, only an
  approved template can be sent. Replying by hand in the WhatsApp or
  WhatsApp Business app isn't limited by it.

## 5. Don'ts

- Don't say or imply that you sent, scheduled or forwarded anything.
- Don't copy more personal data from the screenshot than the reply needs.
- Don't bring up other products or services unless the user asks. If they
  ask whether this can read or send messages by itself, point them to
  `/whatsapp:configure`.
