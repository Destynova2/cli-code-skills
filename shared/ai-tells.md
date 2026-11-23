# AI Tells — Detection and Rewrite

Run this pass on every outbound copy before delivery. The goal is not a
sterile text: it is a text the user could have typed on their phone. Rewrite,
never just delete — a stripped sentence must keep its meaning and rhythm.

## Typographic tells

| Tell | Rewrite direction |
|---|---|
| Em-dash as universal connector ("le projet — qui avance bien — sera...") | Comma, period, colon, or parentheses. In an email or chat, one dash is suspicious, two is a signature. |
| "It's not just X, it's Y" / "ce n'est pas seulement X, c'est Y" | State Y directly. |
| Balanced antithesis chains ("not only... but also...") | Keep one side, drop the scaffolding. |
| Bold words inside a short message | Plain text. Emphasis in a 4-line email is shouting. |
| Bullet list for 2 items or for a simple answer | A sentence. Bullets in chat are a tell on their own. |
| Emoji triplets or one emoji per line | Zero or one emoji, only if the user's own messages use them. |
| Arrows (→) and slash-pairs ("simple/rapide") in prose | Words: "donc", "puis", "ou". |
| Title Case Headings inside an email | No headings in an email under 200 words. |

## Lexical tells — French

| Tell | Rewrite direction |
|---|---|
| "Il est important de noter que" | Delete; say the thing. |
| "En somme", "En résumé" closing a short message | Delete the recap; the message was already short. |
| "Force est de constater" | "On voit que", or delete. |
| "un véritable atout", "une réelle valeur ajoutée" | The concrete fact it stands for. |
| Triads ("simple, rapide et efficace") | Pick the one adjective that is true, or give the fact. |
| "Que ce soit X, Y ou Z" | Name the actual case at hand. |
| "Je me tiens à votre entière disposition" | "Dites-moi si..." with a concrete next step. |
| "N'hésitez pas à..." on every message | Allowed once in a while; replace with a direct offer ("je peux te montrer jeudi"). |
| "Je reviens vers vous dans les meilleurs délais" | A real date or nothing. |
| "Dans un monde en constante évolution" and cousins | Delete the paragraph. |
| Every paragraph opening with a connector ("Par ailleurs", "De plus", "Enfin") | Reorder so the content connects itself. |

## Lexical tells — English

| Tell | Rewrite direction |
|---|---|
| delve, tapestry, testament, landscape, pivotal, crucial | Plain verbs and nouns. |
| comprehensive, robust, seamless, streamline, leverage, foster, elevate | The concrete mechanism or result. |
| "I hope this email finds you well" | Drop, or one specific line ("congrats on the launch"). |
| "I wanted to reach out to..." | Start with the ask. |
| "Please don't hesitate to..." | "Tell me if..." or a dated offer. |
| "As per my last email" | Restate the ask in one fresh line instead. |

## Structural tells

- **The assistant sandwich**: greeting + 3 bullets + recap + offer-to-help
  closer. Break the mold: vary openings, drop the recap, end on the ask.
- **Answering more than asked.** Reply to the question, not to every possible
  question. Extra material goes to a P.S. or a later message, or nowhere.
- **Restating the question before answering.** Start with the answer.
- **Hedging stacks** ("je pense que peut-être on pourrait éventuellement").
  One hedge maximum, and only if uncertainty is real information.
- **Symmetric politeness**: identical greeting/closer pairs on every message
  of a thread. From the second reply in a thread, greetings can drop.
- **Uniform sentence length.** Human email rhythm is irregular: a long
  sentence, then a short one. Vary it.

## Process markers (absolute)

- No AI/model/assistant/prompt mentions, ever, unless the exchange is
  literally about those tools.
- No meta-commentary in or around the copy: "voici un e-mail que vous pouvez
  adapter", "j'espère que ce message conviendra".
- No unfilled placeholders (`[Nom]`, `[date]`, `<COMPANY>`). Fill from
  context or flag `NEEDS-REVIEW` above the copy, outside the paste block.
- No generation artifacts: double spaces, stray markdown in a plain-text
  channel, smart quotes mixed with straight quotes.

## Final check

Read the message aloud once:

- Could the user have typed it in 90 seconds on their phone?
- Does the first line carry the point?
- Is there exactly one ask, and is replying to it cheap?
- Would deleting any sentence lose information? If not, delete it.
