# Collaboration

How agents on this rig communicate with each other. Every crew member and polecat
should know these mechanics.

---

## Communication Channels

### Town Mail

The primary inter-agent messaging channel. Use it for questions, context sharing,
and async collaboration.

```bash
gt mail inbox                                      # Check your messages
gt mail send <rig>/<agent> -s "subject" -m "body"  # Message a specific agent
```

### Nudge

A lightweight tap on the shoulder. Use it for quick pokes — "are you stuck?",
"check your mail," "bead updated with new context."

```bash
gt nudge <rig>/<agent> "message"
```

### Bead Comments

Update the bead itself when the context should travel with the work item. Other
agents watching the bead will see your comment.

```bash
bd update <bead-id> --comment "Added context about..."
```

---

## When to Ask vs. When to File a Bead

Not every question needs a bead, and not every problem needs a conversation.

**Ask another agent when:**
- You have a quick question about their specialty ("Picasso, should this be a modal
  or inline?", "Bob, is this name clear enough?", "Penny, is this path covered?")
- You need a gut check before committing to an approach
- Your bead is tagged `[COLLAB:PM]` and you need project context
- You're unsure whether something is in or out of scope

**File a bead when:**
- The work falls outside your specialty and requires actual implementation
- You've found a bug, security issue, or code smell that someone else should fix
- The effort is more than a quick answer — it's real work that needs tracking

**The line:** If the other agent can answer in a message, ask. If they'd need to open
files and make changes, it's a bead.

---

## Collaboration Etiquette

- **Don't guess at things outside your expertise.** A 30-second question to the right
  person beats 20 minutes of wrong work.
- **Be specific.** "Is this okay?" is a bad question. "I'm rendering the project list
  as cards — should each card link to a detail page or expand inline?" is a good one.
- **Include context.** Reference the bead ID, file path, or specific code you're asking
  about. Don't make the other agent hunt.
- **Check your inbox.** If you're on a `[COLLAB:PM]` bead or expecting a response,
  run `gt mail inbox` regularly.
- **Respect lanes.** Asking for advice is good. Telling another specialist how to do
  their job is not. Picasso owns design decisions. Penny owns test strategy. Bob owns
  code structure. The PM owns project context and routing.
