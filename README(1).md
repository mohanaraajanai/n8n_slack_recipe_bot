# Slack Recipe Ingredient Automation — n8n

## 1. Project Overview

This project is a beginner-to-medium level **n8n automation** that works entirely inside Slack.

A user enters a recipe name in a Slack channel. The n8n workflow receives the Slack message, extracts the recipe name, sends it to an AI/LLM, and posts the required products and ingredients back into the **same Slack channel**.

### Example

**User in Slack:**
```text
Paneer Butter Masala
```

**Bot in Slack:**
```text
Paneer Butter Masala Ingredients:

- Paneer — 250 grams
- Butter — 3 tablespoons
- Oil — 1 tablespoon
- Onion — 1 medium
- Tomatoes — 2 medium
- Ginger-garlic paste — 1 teaspoon
- Cashews — 10–12
- Cream — 3 tablespoons
- Spices — as required
- Fresh coriander — for garnish
```

The complete interaction happens inside Slack. The user does not need to open another application to request or receive the recipe information.

---

## 2. Problem Statement

Build an n8n automation that listens for a recipe name in a Slack channel, asks an AI model to identify the products and ingredients required to make that recipe, and replies with the list in the same Slack channel.

### Difficulty

**Beginner to Medium**

### Tools

- n8n
- Slack
- LLM / AI model
- Slack App / Bot

---

## 3. Objectives

The workflow is designed to:

1. Listen for new messages posted to a Slack channel.
2. Read the recipe name entered by the user.
3. Store the recipe name in a clean field.
4. Preserve the original Slack channel ID.
5. Send the recipe name to an AI model.
6. Generate a short and practical ingredient/product list.
7. Include approximate quantities.
8. Handle unclear recipe names gracefully.
9. Post the AI response back to the same Slack channel.
10. Run automatically using the Slack production webhook.

---

## 4. Workflow Architecture

```text
┌──────────────────────┐
│    Slack Trigger     │
│ New Message Posted   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│     Edit Fields      │
│                      │
│ recipe  = message    │
│ channel = channel ID │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   Basic LLM Chain    │
│                      │
│ Generate ingredients │
│ and quantities       │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   Slack - Send       │
│       Message        │
│                      │
│ Same channel ID      │
└──────────┬───────────┘
           │
           ▼
       Slack Channel
```

---

## 5. n8n Nodes

### 5.1 Slack Trigger

**Purpose:** Start the workflow whenever a new message is posted to the configured Slack channel.

**Trigger:**
```text
New Message Posted to Channel
```

The Slack Trigger provides information including:

- `text` — message entered by the user
- `user` — Slack user information
- `channel` — channel ID
- `ts` — message timestamp
- `team` — Slack workspace/team information

Example:

```json
{
  "type": "message",
  "user": "U0C2HAAEVK2",
  "text": "Paneer Butter Masala",
  "channel": "C0C1P226NV8"
}
```

---

### 5.2 Edit Fields

**Purpose:** Extract and standardize the values required by the remaining workflow.

Two fields are created:

| Field | Expression | Purpose |
|---|---|---|
| `recipe` | `{{ $json.text }}` | Recipe entered by the user |
| `channel` | `{{ $json.channel }}` | Original Slack channel ID |

Example output:

```json
{
  "recipe": "Paneer Butter Masala",
  "channel": "C0C1P226NV8"
}
```

Keeping the channel ID is important because the final Slack node uses it to send the response to the same channel.

---

### 5.3 Basic LLM Chain

**Purpose:** Send the recipe name to an AI model and generate the required ingredients/products.

#### Prompt

```text
List all the products and ingredients needed to make {{ $json.recipe }}.

Return a clear bulleted list with approximate quantities.
Keep it short and practical.

If the recipe name is unclear or empty, respond:
"Please provide a clear recipe name and try again."
```

The expression:

```text
{{ $json.recipe }}
```

is replaced with the recipe captured from Slack.

For example:

```text
List all the products and ingredients needed to make Paneer Butter Masala.
```

The connected Chat Model then generates the response.

---

### 5.4 Slack — Send Message

**Purpose:** Post the AI-generated response back to the original Slack channel.

#### Configuration

**Resource:**
```text
Message
```

**Operation:**
```text
Send
```

**Send Message To:**
```text
Channel
```

**Channel:**
```text
By ID
```

**Channel expression:**
```text
{{ $('Edit Fields').item.json.channel }}
```

**Message Type:**
```text
Simple Text Message
```

**Message Text:**
```text
{{ $json.text }}
```

This ensures that the AI response is posted to the same channel from which the recipe request originated.

---

## 6. Slack Setup

A Slack App/Bot is required to communicate with the n8n workflow.

### Basic setup

1. Create a Slack App.
2. Create/install the bot.
3. Configure the required Slack permissions.
4. Install the app into the Slack workspace.
5. Invite the bot to the test channel.
6. Configure Slack Event Subscriptions.
7. Use the n8n **production webhook URL** for the Slack event request URL.
8. Verify the Slack request URL.
9. Configure the n8n Slack Trigger.
10. Activate the workflow.

### Important

The bot must have permission to receive the required Slack events and send messages to the channel.

---

## 7. Credentials and Security

Credentials should be stored securely in **n8n Credentials**.

Do **not**:

- Put API keys directly inside nodes.
- Commit API keys to GitHub.
- Put Slack tokens in source files.
- Put LLM API keys in README files.
- Share webhook secrets publicly.

The repository should contain only workflow documentation and safe configuration information.

If an exported n8n workflow contains credentials or sensitive values, remove/redact them before committing it to GitHub.

---

## 8. Production Webhook

During development, the Slack Trigger can be tested using n8n's test/listening mode.

For normal operation, the workflow should use the **production webhook** configured in the Slack App's Event Subscriptions.

### Test mode

```text
Click Execute / Listen
        ↓
Send message in Slack
        ↓
Workflow executes
```

### Production mode

```text
User sends message in Slack
        ↓
Slack sends event to production webhook
        ↓
n8n workflow executes automatically
        ↓
AI generates response
        ↓
Bot replies in Slack
```

The production workflow does not require manually clicking **Execute** for every Slack message.

---

## 9. Handling Unclear Recipe Names

The AI prompt includes basic handling for unclear or empty recipe requests.

For example, if the user sends:

```text
xyzabc123
```

the workflow should return:

```text
Please provide a clear recipe name and try again.
```

This provides a graceful response instead of generating an unrelated ingredient list.

---

## 10. Testing

The workflow should be tested with multiple scenarios.

### Test Case 1 — Common recipe

**Input:**
```text
Paneer Butter Masala
```

**Expected result:**

The bot returns a practical list of ingredients and approximate quantities.

---

### Test Case 2 — Another recipe

**Input:**
```text
Chicken Biryani
```

**Expected result:**

The bot returns the products/ingredients required for chicken biryani with approximate quantities.

---

### Test Case 3 — Unclear recipe

**Input:**
```text
xyzabc123
```

**Expected result:**
```text
Please provide a clear recipe name and try again.
```

---

### Test Case 4 — Production test

Send a recipe directly in the configured Slack channel while the n8n workflow is **Active**.

**Expected result:**

The workflow should start automatically without manually executing the workflow from n8n.

---

## 11. Known Consideration — Bot Self-Triggering

The Slack Trigger can receive messages generated by the bot itself.

This can cause a loop such as:

```text
User
  ↓
Recipe request
  ↓
n8n
  ↓
AI
  ↓
Bot response
  ↓
Slack Trigger
  ↓
n8n
  ↓
AI
  ↓
Bot response
  ↓
...
```

### Recommended improvement

For a production-ready implementation, add a filter immediately after the Slack Trigger that identifies and ignores messages generated by the bot.

The exact filtering field should be based on the Slack event payload exposed by the configured n8n Slack Trigger. In this implementation, the trigger output exposes bot/profile information, so the filter should compare the message author against the bot identity rather than assuming that `bot_id` alone is sufficient.

Recommended structure:

```text
Slack Trigger
      ↓
Bot Message Filter
      ↓
   User message?
    /       \
  YES        NO
   ↓          ↓
Edit Fields   Stop
   ↓
Basic LLM Chain
   ↓
Slack Send Message
```

This prevents the bot's own response from being treated as another recipe request.

---

## 12. Example End-to-End Flow

### Step 1 — User

The user enters:

```text
Masala Dosa
```

in the Slack channel.

### Step 2 — Slack Trigger

The Slack event is received by n8n.

```text
text = Masala Dosa
channel = C0C1P226NV8
```

### Step 3 — Edit Fields

The data is simplified:

```json
{
  "recipe": "Masala Dosa",
  "channel": "C0C1P226NV8"
}
```

### Step 4 — Basic LLM Chain

The AI receives the recipe name and generates an ingredient list.

### Step 5 — Slack Send Message

n8n sends the AI response to:

```text
C0C1P226NV8
```

### Step 6 — User sees the response

The final interaction remains inside Slack.

---

## 13. Benefits

- Entire interaction happens in Slack.
- No separate recipe application is required.
- AI generates practical ingredient lists.
- Approximate quantities are included.
- The original channel is preserved.
- The workflow can run automatically using a production webhook.
- The solution can be extended to support additional recipe-related functionality.

---

## 14. Future Enhancements

Possible improvements include:

- Add serving-size support.
- Ask the user whether they want vegetarian or non-vegetarian ingredients.
- Generate a shopping checklist.
- Categorize ingredients into vegetables, dairy, spices, etc.
- Support multiple recipes in one request.
- Add dietary preferences.
- Add cuisine selection.
- Format responses using Slack Block Kit.
- Add bot-message filtering to prevent self-trigger loops.
- Store frequently requested recipes.
- Add conversation/context support.

---

## 15. Project Requirements Checklist

| Requirement | Status |
|---|---|
| Use Slack Trigger | ✅ Completed |
| Read recipe name from Slack | ✅ Completed |
| Pass recipe to AI/LLM | ✅ Completed |
| Generate products/ingredients | ✅ Completed |
| Include approximate quantities | ✅ Completed |
| Reply in the same Slack channel | ✅ Completed |
| Handle unclear recipe name | ✅ Completed |
| Use production webhook | ✅ Tested |
| Automatic execution | ✅ Tested |
| Bot self-trigger protection | ⚠️ Recommended enhancement |

---

## 16. Conclusion

This project demonstrates a simple AI-powered automation using **Slack + n8n + an LLM**.

The user interacts entirely through Slack:

```text
Ask in Slack
    ↓
n8n receives request
    ↓
AI processes recipe
    ↓
n8n formats response
    ↓
Bot replies in Slack
```

The workflow demonstrates key automation concepts including event triggers, data mapping, expressions, AI integration, API/webhook communication, and automated responses.
