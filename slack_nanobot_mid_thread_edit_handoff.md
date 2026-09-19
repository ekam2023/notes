# Technical Handoff: Mid-Thread Edit Support in Slack for Nanobot

## 1. Context & Motivation
In standard web interfaces (e.g., ChatGPT/Claude), editing an earlier message truncates downstream conversation turns and regenerates context from that point forward. 

In Nanobot, conversation state for Slack is persisted in thread-scoped append-only JSONL files (`session_key = slack:{channel_id}:{thread_ts}`). Nanobot does not currently listen for `message_changed` events, meaning edits in Slack are ignored by the backend session store and the active context.

This document outlines a **minimal-risk, non-breaking approach** to support mid-thread message edits and regeneration while preserving prompt caching efficiency.

---

## 2. Core Architecture & Workflow

```
[User edits Slack message at T_edit]
                 │
                 ▼
[Slack Event API: subtype == "message_changed"]
                 │
                 ▼
[SlackChannel Handler: verify message author & target thread]
                 │
                 ▼
[SessionManager: rewind_to_timestamp(T_edit) or replace_turn(T_edit)]
                 │
                 ▼
[Truncate/Rewrite Session JSONL from T_edit onward]
                 │
                 ▼
[Trigger Agent Execution Loop on revised context]
                 │
                 ▼
[Slack UI: chat.update bot reply at T_edit+1 AND/OR chat.delete downstream bot turns]
```

---

## 3. Proposed Minimal-Risk Code Changes

### Step 1: Add Event Handling in Slack Channel (`nanobot/channels/slack.py`)
Subscribe to `message_changed` events in Socket Mode / Events handler:

```python
@slack_app.event("message")
async def handle_message_events(event, say, client):
    subtype = event.get("subtype")
    
    # Handle mid-thread user edit
    if subtype == "message_changed":
        message_data = event.get("message", {})
        thread_ts = message_data.get("thread_ts")
        
        # Only process edits within threaded conversations
        if not thread_ts:
            return
            
        edited_ts = message_data.get("ts")
        new_text = message_data.get("text")
        user_id = message_data.get("user")
        channel_id = event.get("channel")
        
        session_key = f"slack:{channel_id}:{thread_ts}"
        
        await agent.handle_message_edit(
            session_key=session_key,
            channel_id=channel_id,
            thread_ts=thread_ts,
            edited_ts=edited_ts,
            new_text=new_text,
            user_id=user_id,
            client=client
        )
```

---

### Step 2: Extend Session Manager (`nanobot/session/manager.py`)
Add a deterministic rewind method to modify the persisted session JSONL file without corrupting unrelated sessions.

```python
def rewind_and_update(self, session_key: str, target_ts: str, updated_content: str) -> list:
    """
    Truncates all turns strictly after target_ts, updates target_ts turn content,
    and returns the new valid context messages.
    """
    session = self.get_or_create(session_key)
    updated_history = []
    
    found = False
    for msg in session.messages:
        # Match message by timestamp metadata
        msg_ts = msg.metadata.get("ts")
        if msg_ts == target_ts:
            msg.content = updated_content
            updated_history.append(msg)
            found = True
            break  # Truncate all subsequent messages (GPT chat behavior)
        else:
            updated_history.append(msg)
            
    if found:
        session.messages = updated_history
        self._persist_session(session_key, session)
        
    return session.messages
```

---

### Step 3: Slack Cleanup & Bot Response (`nanobot/agent/core.py`)
When regenerating after an edit:
1. **Locate Downstream Turns**: Query Slack thread for bot messages posted after `target_ts`.
2. **In-Place Update (`chat.update`)**: Update the immediate assistant response to reflect the new output.
3. **Prune Stale Replies (`chat.delete`)**: Delete subsequent bot messages belonging to discarded branches.

---

## 4. Token Caching Considerations
To preserve KV prefix caching across edits:
- **Prefix Isolation**: Keep static system prompts, MCP schemas, and unedited parent messages identical byte-for-byte.
- **Cache Breakpoints**: The prompt prefix *up to `target_ts`* will hit the cache; only the edited turn and subsequent generation will incur prompt token recomputation.

---

## 5. Risk Assessment & Mitigations

| Risk | Impact | Mitigation Strategy |
| :--- | :--- | :--- |
| **Race Conditions on Fast Edits** | Inconsistent session state | Introduce a per-thread asyncio lock (`asyncio.Lock`) on session writes. |
| **Accidental Bot-Edit Loop** | Infinite trigger loops | Ignore `message_changed` events where `message.bot_id` or `message.user` matches the bot's own ID. |
| **Slack API Rate Limits** | `chat.delete` throttling | Limit deletion to the bot's own immediate responses; avoid bulk-deleting large threads. |
| **Shared Workspace Edits** | Overwriting other users' context | Verify `user_id` matches the original sender of `target_ts` before rewinding context. |
