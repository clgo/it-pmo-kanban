# Hook: chatbot-voice (v2 website)

| | |
|---|---|
| **Trigger** | The user clicks the cyberpunk avatar chatbot launcher (bottom-right). |
| **Action** | Opens the chat panel and speaks: *"I am your IT assistant, how can I help you?"* (`VOICE_GREETING`). |
| **Code** | `index.html` → `initChatbot()` (script section "HOOKS (v2)"); markup `#cb`; styles section 9; avatar is the inline SVG `<symbol id="cb-avatar">`. |

## Voice

- Uses the browser's built-in Web Speech API (`speechSynthesis`), so there are no audio files, no external service and no network call.
- It runs inside the click handler, which browsers require before they allow speech.
- The speaker button in the panel header mutes the voice; closing the panel cancels speech in progress.
- If the browser has no speech support, the hook does nothing and the chat still works.

## Text RAG

Retrieval-augmented answers, entirely client-side:

1. **Corpus (rebuilt on every question):** a static knowledge base `CB_KB` (policies, how-tos, meeting), a live *Board summary* (counts, overdue IDs, open tasks per assignee) and one document per current task.
2. **Retrieve:** the question is tokenised (stop-words removed, simple plural stemming) and every document is scored with BM25; the top 3 are kept.
3. **Generate:** the reply quotes the retrieved passages and labels each with its source. Below the confidence threshold it says it has no match rather than guessing.

All text is written with `textContent`, so task titles and descriptions can never inject markup.

## Extend

Add entries to `CB_KB` to teach it new topics. Swapping in a real LLM later would only replace `cbAnswer()`; keep the retrieval step so answers stay grounded.
