# The Complete Guide to DeepSeek Tool Calling in 2026

DeepSeek supports tool calling through its API, allowing applications to give the model access to external functions and services. The model can decide when a registered function is needed, generate structured arguments, and return the tool call to your application. Your application then executes the function and sends the result back to the model so it can produce the final response.

This makes DeepSeek highly effective for AI agents, automation systems, database applications, API integrations, coding assistants, and data-processing workflows.

This guide explains how DeepSeek tool calling works, how to implement it in Python, and how to fix common local deployment issues like infinite tool loops and DSML tag leaks.

---

## Does DeepSeek Support Tool Calling?

Yes. DeepSeek's API supports function/tool calling.

DeepSeek's current API documentation includes tool-calling support for its current API models, including DeepSeek-V4-Flash and DeepSeek-V4-Pro. Older names such as deepseek-chat and deepseek-reasoner are being phased toward the newer V4 model generation.

DeepSeek's API is also designed to be compatible with the common OpenAI-style chat-completions interface, which makes migration easier for applications already using compatible APIs.

---

## What Is Tool Calling?

Tool calling allows an AI model to request that your application execute a predefined function.

For example, imagine you give the model a tool called: `get_crypto_price`

The user asks: *What is the current price of Bitcoin?*

Instead of inventing a number, the model can request:

```json
{
  "name": "get_crypto_price",
  "arguments": {
    "symbol": "BTC"
  }
}
```

Your application receives that request, executes the real API call, and returns the result to DeepSeek.

The important distinction is: **DeepSeek generates the tool call; your application executes the tool.** The model does not automatically gain unrestricted access to your computer, database, shell, or APIs.

---

## How DeepSeek Tool Calling Works

A typical tool-calling workflow looks like this:

```plaintext
User ➔ DeepSeek model ➔ Tool call generated ➔ Your application ➔ External function/API/database ➔ Tool result ➔ DeepSeek model ➔ Final response
```

There are three important components:

1. **Model:** The model decides whether one of the available tools is appropriate for the user's request.
2. **Tool definition:** Your application describes each available function using a name, description, and parameter schema.
3. **Tool executor:** Your application receives the generated call, validates the arguments, runs the function, and returns the result.

---

## Example: Defining a Tool

Here is a simple currency-conversion tool definition:

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "convert_currency",
            "description": "Convert an amount from one currency to another.",
            "parameters": {
                "type": "object",
                "properties": {
                    "from_currency": {
                        "type": "string",
                        "description": "Source currency code, for example USD"
                    },
                    "to_currency": {
                        "type": "string",
                        "description": "Target currency code, for example EUR"
                    },
                    "amount": {
                        "type": "number",
                        "description": "Amount to convert"
                    }
                },
                "required": [
                    "from_currency",
                    "to_currency",
                    "amount"
                ]
            }
        }
    }
]
```

The tool definition tells DeepSeek what the function does and what arguments it expects.

---

## Python Example

DeepSeek provides an OpenAI-compatible API style, so a Python application can use a familiar client pattern.

```python
from openai import OpenAI
import json

client = OpenAI(
    api_key="YOUR_DEEPSEEK_API_KEY",
    base_url="https://api.deepseek.com"
)

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_crypto_price",
            "description": "Get the current cryptocurrency price.",
            "parameters": {
                "type": "object",
                "properties": {
                    "symbol": {
                        "type": "string",
                        "description": "Cryptocurrency symbol, such as BTC"
                    }
                },
                "required": ["symbol"]
            }
        }
    }
]

messages = [
    {
        "role": "user",
        "content": "What is the current price of BTC?"
    }
]

response = client.chat.completions.create(
    model="deepseek-v4-flash",
    messages=messages,
    tools=tools
)

message = response.choices[0].message
print(message)
```

---

## Troubleshooting Local Deployments: Fixing Tool Loops and DSML Leaks

When running DeepSeek models locally using frameworks like **vLLM**, **Ollama**, or custom setups, developers frequently run into issues where the model starts looping tool calls, repeating shell commands, or leaking raw **DSML (DeepSeek Markup Language)** tags (such as `<｜tool calls｜>`, `<｜action｜>`, or `<｜thought｜>`) into final outputs.

Here is how to resolve these issues.

### 1. Fixing Speculative Decoding Issues in vLLM

If speculative decoding is enabled or misconfigured in local vLLM deployments, token-sampling mismatches can occur during tool-call generation. This leads to recursive loops where the model repeats the same function call.

To fix this, ensure you do not use draft/speculative model options unless verified, and run vLLM in eager execution mode to ensure deterministic sampling:

```bash
python -m vllm.entrypoints.openai.api_server \
  --model deepseek-ai/DeepSeek-V4-Flash \
  --enforce-eager \
  --tensor-parallel-size 1 \
  --dtype float16
```

### 2. Fixing Loop Conditions in Ollama (Modelfile)

Ollama defaults can cause DeepSeek to drift during multi-turn function calling. To stabilize formatting and prevent looping, modify your custom `Modelfile` to include low-latency constraints and strict stop tokens:

```yaml
FROM deepseek-v4-flash

# Enforce zero temperature for deterministic tool calling
PARAMETER temperature 0.0
PARAMETER top_p 0.95

# Set custom stop tokens to prevent the model from leaking internal DSML segments
PARAMETER stop "<｜tool calls｜>"
PARAMETER stop "<｜action｜>"
PARAMETER stop "｜/tool calls｜>"
```

Build the custom model:
```bash
ollama create deepseek-stable -f ./Modelfile
```

### 3. Implementing a Stream Repair Parser in Python

During active streaming, DSML blocks can arrive fragmented. If your client parser fails to handle incomplete tags, the invalid JSON causes retries and loops.

Use this regex-based decorator to clean up incoming tokens and ensure tool-call boundaries match DeepSeek's native separator syntax (`｜`):

```python
import re
import json
from functools import wraps

def repair_deepseek_stream(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        buffer = ""
        repaired_output = ""

        def handle_chunk(chunk_text):
            nonlocal buffer, repaired_output
            buffer += chunk_text

            # Match native DeepSeek V4 DSML tool calling wrappers
            pattern = r"<｜tool calls｜>(.*?)｜/tool calls｜>"
            matches = re.findall(pattern, buffer, flags=re.DOTALL)

            for match in matches:
                repaired_output += f"<｜tool calls｜>{match}｜/tool calls｜>\n"
            
            buffer = re.sub(pattern, '', buffer, flags=re.DOTALL)
            return [f"<｜tool calls｜>{m}｜/tool calls｜>" for m in matches]

        yield from func(*args, **kwargs, on_chunk=handle_chunk)
    return wrapper
```

### 4. Preventing Prompt History Poisoning

If reasoning traces or internal planner text (such as `<｜thought｜>` or `<｜Planner｜>`) are replayed into subsequent turns, the context window gets cluttered with stale data, causing the model to hallucinate or get stuck. 

Aggressively prune internal tags from assistant turns before sending the conversation history back to the model:

```python
def strip_internal_reasoning(message_str: str) -> str:
    patterns = [
        r"<｜Planner｜>.*?｜Plan end｜>",
        r"<｜thought｜>.*?｜/thought｜>",
        r"<｜action｜>.*?｜/action｜>"
    ]
    for pat in patterns:
        message_str = re.sub(pat, "", message_str, flags=re.DOTALL)
    return message_str

# Clean up conversation logs
for msg in conversation_history:
    if msg['role'] == 'assistant':
        msg['content'] = strip_internal_reasoning(msg['content'])
```

---

## Security Considerations

* **Validate every argument:** Never trust model-generated arguments. Check data types, ranges, allowed values, and string lengths before running functions.
* **Expose narrow tools:** Avoid generic executors like `run_shell(cmd)`. Instead, expose restricted functions like `read_file_safe(path)` or `send_email_notification(to, body)`.
* **Enforce external authorization:** The model should not dictate permissions. Check user privileges at the application level before executing any tool calls.

---

## Frequently Asked Questions

* **Is DeepSeek tool calling free?** Tool calling is an API feature, but total cost is based on token usage. Check DeepSeek's pricing page for current V4 rates.
* **Does DeepSeek support strict mode?** Yes. DeepSeek's beta endpoint supports `strict: true` schemas to force JSON compliance.
* **Can I run tool calling locally?** Yes, as long as your inference engine (vLLM/Ollama) supports function schemas and your wrapper application handles execution.

---

## Official Sources

* DeepSeek API documentation: [https://api-docs.deepseek.com/](https://api-docs.deepseek.com/)
* DeepSeek Tool Calls guide: [https://api-docs.deepseek.com/guides/tool_calls](https://api-docs.deepseek.com/guides/tool_calls)
* DeepSeek API reference: [https://api-docs.deepseek.com/api/create-chat-completion](https://api-docs.deepseek.com/api/create-chat-completion)

<script type="application/ld+json">
{"@context": "https://schema.org", "@type": "Article", "headline": "The Complete Guide to DeepSeek Tool Calling in 2026", "description": "Learn how DeepSeek tool calling works, how to implement it in Python, and how to fix local vLLM and Ollama tool loops or DSML leaks.", "datePublished": "2026-08-21", "dateModified": "2026-08-21", "author": {"@type": "Person", "name": "AInside", "url": "https://medium.com/@muhamedfazalps7"}, "publisher": {"@type": "Organization", "name": "AInside"}, "mainEntityOfPage": "https://newchannelid432-code.github.io/seo-blog/deepseek-tool-calling.html"}
</script>
<script type="application/ld+json">
{"@context": "https://schema.org", "@type": "FAQPage", "mainEntity": [{"@type": "Question", "name": "Is DeepSeek tool calling free?", "acceptedAnswer": {"@type": "Answer", "text": "Tool calling itself is an API capability, but the total cost depends on the model and API usage. Check DeepSeek's current pricing documentation for the latest rates."}}, {"@type": "Question", "name": "Does DeepSeek support strict mode?", "acceptedAnswer": {"@type": "Answer", "text": "Yes. DeepSeek's beta endpoint supports strict: true schemas to force JSON compliance."}}, {"@type": "Question", "name": "Can I run tool calling locally?", "acceptedAnswer": {"@type": "Answer", "text": "Yes, as long as your inference engine (vLLM/Ollama) supports function schemas and your wrapper application handles execution."}}]}
</script>
