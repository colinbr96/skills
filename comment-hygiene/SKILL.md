---
name: comment-hygiene
description: >-
  Tightens comments and documentation in pending changes by removing obvious
  restatements, historical asides, fluff, and AI mannerisms.
---

Review comments and documentation for this session. Edit in place. Don't change code behavior. Leave comments and docs that already satisfy these rules untouched.

### Don't restate the obvious

If a piece of code has self-evident behavior or is already explained by another comment or piece of documentation, remove all extraneous text.

### Don't reference prior functionality

Anyone reading the code is looking for how it works today. They don't need a history lesson; if they need to explore previous changes, they can check the Git history.

### Reduce fluff

Use simplified technical English when sufficient to convey your ideas. The fewer words the next person has to read, the easier the code is to understand and the more time-efficient they can be.

### Include the why

If an explanation is warranted, include the justification behind the logic, not just what it does. This can be informal: "Here we X because Y."

### Don't call out the absence of functionality

Comments should explain what the code does, not what it doesn't do. Focus on the positive intent and behavior.

### Avoid common AI mannerisms

- Don't use "it's not X, it's Y" phrasing
- Don't leave out the "that": "The setting the user selected" → "The setting that the user selected".

### Be careful with nouns that read as verbs

Words like update, record, set, change, request, etc., can sound like either a noun or a verb, so terse comments become ambiguous. Add an article or rephrase so only one interpretation makes sense.

