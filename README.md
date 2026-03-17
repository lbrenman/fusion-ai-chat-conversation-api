# Amplify Fusion AI Chat Conversation Readme

A simple, stateful AI chat API built Amplify Fusion that maintains conversation context across multiple turns using PostgreSQL (Neon) for message history storage. The API is OpenAI compliant and supports optional conversation ID management by Fusion. Groq is the LLM used in this example, but any of the Amplify Fusion AI Connectors can be used.

## API

### POST /v1/prompt

Submit a prompt and receive a response from the LLM with full conversation context.

#### Request Body

```json
{
  "prompt": "What is my name?",
  "conversationId": "abc-123"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `prompt` | string | Yes | The user's message to send to the LLM |
| `conversationId` | string | No | ID of an existing conversation. If omitted, a new conversation is created |

#### Response Body

```json
{
  "message": "Your name is Leor.",
  "conversationId": "abc-123"
}
```

| Field | Type | Description |
|---|---|---|
| `message` | string | The assistant's reply |
| `conversationId` | string | The conversation ID (use this in subsequent requests to maintain context) |

#### Error Response

```json
{
  "message": "Error description"
}
```

## How It Works

Each turn follows this flow:

```
Client sends prompt + optional conversationId
       ↓
1. Upsert user message into DB
       ↓
2. Fetch full message array from DB
       ↓
3. Send full message array to OpenAI
       ↓
4. Upsert assistant response into DB
       ↓
5. Return response + conversationId to client
```

By passing the full message history to OpenAI on every request, the model has complete conversational context — allowing it to answer questions like "What is my name?" based on something the user said several turns earlier.

## Database

### PostgreSQL Schema

```sql
CREATE TABLE conversations (
  conversation_id  VARCHAR(255) PRIMARY KEY,
  created_at       TIMESTAMP DEFAULT NOW(),
  updated_at       TIMESTAMP DEFAULT NOW(),
  messages         JSONB NOT NULL DEFAULT '[]'
);
```

### Upsert Message

```sql
INSERT INTO conversations (conversation_id, messages)
VALUES (:conversationId, CAST(:messageArray AS jsonb))
ON CONFLICT (conversation_id)
DO UPDATE SET
  messages = CAST(:messageArray AS jsonb),
  updated_at = NOW();
```

### Retrieve Message History

```sql
SELECT messages FROM conversations WHERE conversation_id = :conversationId;
```

### Delete Old Conversations

```sql
DELETE FROM conversations
WHERE updated_at < NOW() - INTERVAL '24 hours';
```

Change '24 hours' to whatever TTL makes sense for your use case, e.g. '7 days', '1 hour', etc.

A schedule triggered integration is included for cleanup using the `Delete Old Conversations` SQL above

## Conversation ID Management

The client is responsible for managing conversation IDs.

- If no `conversationId` is provided, the server generates a new one and returns it in the response
- Pass the returned `conversationId` in all subsequent requests to maintain context
- A null or missing `conversationId` starts a fresh conversation with no prior context

## Testing Context

A simple way to verify that conversation context is working correctly:

1. Send: `"My name is Leor."` → start a new conversation, save the returned `conversationId`
2. Send: `"What is 2 + 2?"` → pass the same `conversationId`
3. Send: `"What is my name?"` → pass the same `conversationId`

**Expected response:** `"Your name is Leor."`

If context is NOT being used, the model will respond that it doesn't know your name.

## OpenAPI Spec

See [chat.yaml](./chat.yaml) for the full OpenAPI 3.0 spec.