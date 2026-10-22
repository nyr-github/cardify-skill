---
name: cardify-skill
description: Converts Markdown into a Cardify knowledge card and returns an editable link plus a read-only preview. Use when the user wants to turn Markdown notes, an article, or a list into a visual card or poster (Xiaohongshu, Instagram, TikTok style), asks for a Cardify link, or wants to preview and restyle a card online. Trigger phrases include "turn Markdown into a card", "turn this article or note into a card", "Markdown to card", "generate a card image", "knowledge card", "Xiaohongshu-style card" and "Cardify link". When the user expresses any intent to turn Markdown, an article, or a note into a card, treat it as a command — run the workflow and return the links instead of explaining how.
---

# Cardify Card Link

Turn Markdown into a Cardify card and give the user a link that opens it in the Cardify editor. No API key or account is required.

## When to use

- The user says anything close to "turn Markdown into a card": make a card out of this article, these notes, or this list; generate a card image; or produce a Xiaohongshu-style post. Treat this as a command — run the workflow and return the links, do not reply with instructions.
- The user wants Markdown notes turned into a visual card, poster, or social post.
- The user asks how to preview, restyle, or export a Markdown card.
- The user mentions Cardify or a Xiaohongshu-style image card, or asks for an editable card link.

## Workflow

1. Prepare the Markdown.
2. Choose the optional card settings.
3. POST to the Cardify API.
4. Return the `editUrl` (and `shareUrl`) to the user.

### Step 1: Prepare the Markdown

- Use the user's content as-is; fix only obvious Markdown breakage.
- The first `#` heading becomes the card title. Keep it short.
- A line containing only `---` starts a new card in `line` mode.
- Keep paragraphs short; prefer lists over long blocks of text.
- Images must be public URLs, not local file paths.
- The API accepts up to 100,000 characters.
- Remove secrets, tokens, or private data before sending content to the API.

### Step 2: Create the card

Write the Markdown to a temporary file, then POST it as the raw request body:

```bash
curl -sS -X POST "https://cardify.aivaded.com/api/card-link?locale=en&theme=card-ins" \
  -H "Content-Type: text/markdown; charset=utf-8" \
  --data-binary @card.md
```

Add options as query parameters when needed:

```bash
curl -sS -X POST "https://cardify.aivaded.com/api/card-link?locale=en&theme=card-ins&size=post_instagram" \
  -H "Content-Type: text/markdown; charset=utf-8" \
  --data-binary @card.md
```

JSON request bodies also work:

```bash
curl -sS -X POST "https://cardify.aivaded.com/api/card-link" \
  -H "Content-Type: application/json" \
  -d '{"markdown":"# Title\n\nBody text","locale":"en","theme":"card-ins"}'
```

To upload a Markdown file instead, send it as multipart form data:

```bash
curl -sS -X POST "https://cardify.aivaded.com/api/card-link" \
  -F "file=@note.md" \
  -F "locale=en" \
  -F "theme=card-ins"
```

If the environment has no shell, send the same request with any HTTP client.

The response is JSON:

```json
{
  "id": "9ed5afeb0b3b89f0d19c74c244c65156",
  "locale": "en",
  "editUrl": "https://cardify.aivaded.com/en/editor/9ed5afeb0b3b89f0d19c74c244c65156",
  "shareUrl": "https://cardify.aivaded.com/en/share/9ed5afeb0b3b89f0d19c74c244c65156",
  "theme": "card-ins",
  "size": "post_instagram",
  "mode": "expand",
  "width": 440,
  "height": 550
}
```

### Step 3: Reply to the user

Give the links in this form:

```text
Editable card: <editUrl>
Preview: <shareUrl>
```

Explain briefly:

- The editor link opens the card with the Markdown already loaded, so the user can change the theme, size, and fonts.
- No account is needed to edit; signing in syncs the card across devices.
- Export to PNG, SVG, or PDF is a paid feature on cardify.aivaded.com. The API itself only creates the card link.

## Options

| Field | Values | Notes |
| --- | --- | --- |
| `locale` | `zh`, `en` | Match the content language. `zh` links have no prefix; `en` links use `/en`. |
| `theme` | `theme_class`, e.g. `card-ins`, `card-muji`, `card-business`, `card-xiaohongshu` | Recommended. Use `card-muji` for Chinese and `card-ins` for English when the user has no preference. |
| `size` | `post_xiaohongshu`, `poster_xiaohongshu`, `post_instagram`, `post_tiktok`, `square`, `a4` | Defaults: `post_xiaohongshu` for zh, `post_instagram` for en. |
| `mode` | `expand`, `line`, `split`, `hide` | `expand` = one long card (default); `line` = split on `---`; `split` = equal height; `hide` = crop to fixed height. |
| `font` | font key | Omit unless the user asks for a specific font. |
| `scale` | 50 to 300 | Render scale in percent. Defaults to 100. |
| `width`, `height` | 200 to 3000 | Custom canvas size in px. Use only when an exact size is required. |

### Choosing a theme

Pass a theme explicitly. If the user has no preference, use `card-muji` for Chinese and `card-ins` for English:

| Content | Suggested theme |
| --- | --- |
| Chinese lifestyle, notes, reading | `card-muji` |
| Chinese study, tutorials, checklists | `card-study` or `card-notebook` |
| Chinese social posts for Xiaohongshu | `card-xiaohongshu` |
| Business or reports | `card-business` |
| English, general purpose | `card-ins` |
| English, high contrast or tech | `card-darktech` |

If the user asks for the full theme list, call `list_card_options` (see below) instead of guessing.

## Discover all options

The MCP endpoint exposes the complete theme, size, mode, and font list:

```bash
curl -sS "https://cardify.aivaded.com/api/mcp" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"list_card_options","arguments":{}}}'
```

## Handle errors

- **400**: the response `error` field says what is wrong (usually an unknown theme or size). Retry once without `theme` and `size`, or with values from `list_card_options`.
- **429**: the API allows 60 cards per hour per IP. Tell the user to wait; do not loop.
- **5xx or network error**: retry once after a short delay, then report the failure.
- **Missing `editUrl`**: treat the call as failed. Never invent a link.

## MCP alternative

Agents with MCP support can call the same service directly:

- Endpoint: `https://cardify.aivaded.com/api/mcp` (Streamable HTTP, stateless JSON-RPC, no API key)
- Tools: `create_card_link` (same inputs as the HTTP API) and `list_card_options`

```bash
curl -sS "https://cardify.aivaded.com/api/mcp" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"create_card_link","arguments":{"markdown":"# Hello Cardify\n\nTurn Markdown into a card.","locale":"en","theme":"card-ins"}}}'
```

## Rules

- Only create cards the user asked for. Do not upload unrelated content.
- `editUrl` and `shareUrl` are public-by-URL but unguessable. Do not post them publicly without the user's consent.
- The API returns links, not image files. Do not claim a PNG, SVG, or PDF was generated.
- Do not promise features beyond the Cardify editor. The product guide is at `https://cardify.aivaded.com/llms.txt`.
