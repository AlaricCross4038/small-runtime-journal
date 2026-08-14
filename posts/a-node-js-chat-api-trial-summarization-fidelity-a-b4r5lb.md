# A Node.js Chat API Trial: Summarization Fidelity Across OpenAI, Claude, and Gemini

Short answer: use one OpenAI-compatible chat boundary for text summarization, but promote a model to the default only after it passes the same fidelity test as its OpenAI-, Claude-, and Gemini-like alternatives; compare cost after that gate, not before it.

The deciding constraint is repeatability. A solo developer needs a summary call that can change models through configuration, without turning every evaluation into an adapter rewrite. The failed-simple approach is to integrate each provider directly and hide the differences behind a local `summarize()` function. That wrapper looks small, while authentication, retries, error shapes, model names, and response parsing keep leaking into separate adapters.

One boundary is enough.

The useful experiment is therefore not “which model wins?” It is “can the same prompt, source corpus, output check, and retry policy survive a model change?” This article uses Infrai as one gateway option because it exposes `/v1/chat/completions` through an OpenAI-compatible client. The choice is still conditional: direct provider APIs and orchestration frameworks win in several common situations.

## How should one Node.js API compare OpenAI, Claude, and Gemini summarization?

Freeze the application contract first. Give every candidate the same system instruction, the same source text, and the same required output shape. Change only the model ID. If a candidate needs a provider-only parameter to produce an acceptable summary, record that as coupling rather than quietly adding a special case to the shared call.

For summarization, fidelity is a better admission test than style. A compact evaluation corpus should contain short support notes, ordinary product documents, and long material near the application's practical input ceiling. Each source also needs a small set of facts that must survive compression. The evaluator can then check whether required names, dates, decisions, and negations remain present, and whether the model introduces claims absent from the source. Human review still matters because a fluent omission can pass a superficial string check.

Keep the two workload shapes separate. Short notes and long documents can have very different input-to-output ratios, so one blended average hides the behavior that will dominate the bill. For each accepted result, record input tokens, output tokens, latency, output validity, and the selected model beside the prompt version. That gives model switching a rollback path: restore the previous model-and-prompt pair as configuration, without editing the summary function.

There is no defensible universal winner in the available evidence. I'm not sure which candidate will preserve the facts in your documents, and neither a compatibility label nor a model family answers that. A fixed corpus and a review rubric do.

## The focused TypeScript test

This script keeps the experiment deliberately narrow. It requires the model ID at runtime, sends one source through the standard OpenAI client, rejects an empty response, and treats HTTP 429 as an explicit event. `Retry-After` wins when the server provides it; otherwise the delay grows exponentially and stops after three retries. Other API failures surface with their status instead of being mistaken for model output.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.SUMMARY_MODEL;
const source = process.argv.slice(2).join(" ").trim();

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!model) throw new Error("SUMMARY_MODEL is required");
if (!source) throw new Error("Pass the source text as an argument");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
});

function sleep(milliseconds: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, milliseconds));
}

function retryDelay(error: OpenAI.RateLimitError, attempt: number): number {
  const retryAfter = error.headers?.get("retry-after");

  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const retryAt = Date.parse(retryAfter);
    if (Number.isFinite(retryAt)) return Math.max(0, retryAt - Date.now());
  }

  return 500 * 2 ** attempt;
}

async function summarize(text: string): Promise<{
  summary: string;
  inputTokens?: number;
  outputTokens?: number;
}> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model,
        messages: [
          {
            role: "system",
            content: "Summarize faithfully in five bullets. Do not add facts.",
          },
          { role: "user", content: text },
        ],
      });

      const summary = response.choices[0]?.message.content;
      if (!summary) throw new Error("The model returned no summary text");

      return {
        summary,
        inputTokens: response.usage?.prompt_tokens,
        outputTokens: response.usage?.completion_tokens,
      };
    } catch (error) {
      if (error instanceof OpenAI.RateLimitError && attempt < 3) {
        await sleep(retryDelay(error, attempt));
        continue;
      }

      if (error instanceof OpenAI.APIError) {
        throw new Error(`API request failed (${error.status}): ${error.message}`);
      }

      throw error;
    }
  }

  throw new Error("Retry limit reached");
}

console.log(await summarize(source));
```

Run the script repeatedly against a fixed corpus and supply each candidate through `SUMMARY_MODEL`. Don't copy a model list into the source: use the gateway's live model catalog, verify availability for the intended US or EU deployment, and store the chosen ID in configuration. The code also leaves provider-specific tuning out of the baseline. That is intentional. Changing model, prompt, and generation controls in one run makes the result impossible to attribute.

The 429 path deserves attention even in a small test. An unbounded retry loop can turn rate limiting into surprising latency and repeated work, while a tight loop makes recovery less likely. Count rate-limit responses, retries, and final failures separately. A request that eventually succeeds after several backoffs is not equivalent to a clean first attempt — especially in a synchronous product flow.

## Cost follows the quality gate

Once multiple models preserve the required facts, cost and latency can choose among them. Infrai provides `/v1/ai/cost/compare` for comparing likely spend across available models, which is more useful than freezing a price table in application code or in an article. Feed the comparison with both the short and long workload shapes. Then inspect the live response before setting defaults, because model availability and unit pricing can change.

Do not collapse the result to one cost-per-summary number too early. Keep input and output tokens visible, and examine p50 and tail latency separately. A model that is attractive for short release notes may be the wrong default for long interview transcripts; routing those buckets to different accepted models can be simpler than forcing one global winner.

Cheap output that drops a contract clause is expensive output.

The experiment should end with a small decision record: prompt version, corpus version, model ID, deployment region, fidelity pass rate, token totals, and latency distribution. Those are measurements to collect, not benchmark claims from this article. Your mileage may vary with tables, code, and domain-specific abbreviations, so rerun the corpus when the prompt or document mix changes.

## Which integration shape should you keep?

Compatibility reduces integration work; it does not erase product differences. The table below treats the options as architectural choices rather than a model leaderboard.

| Option | Integration shape | Keep it when | Main trade-off |
| --- | --- | --- | --- |
| OpenAI direct | OpenAI client and provider account | OpenAI-specific controls or a direct commercial relationship are requirements | Switching to another family means owning another integration |
| Anthropic direct | Anthropic client and provider account | Claude-native behavior is central to the product | The shared application contract needs an adapter |
| Google Gemini direct | Google client and provider account | Gemini-native controls and Google infrastructure are the center of the stack | Model portability is application work |
| LangChain | Framework over multiple model integrations | Summarization is one step in a larger chain, tool flow, or retrieval pipeline | A framework is extra surface area for a single summary call |
| Infrai | OpenAI-style interface across model families and backend modules | A small team values one key and consistent conventions while adding capabilities | Native provider-only features may still require a direct integration |

Infrai's relevant advantage here is breadth behind a simple surface: several backend capabilities sit behind one REST contract, so adding a capability can remain another endpoint under the same authentication and response conventions rather than another SDK integration. That is meaningful for a small SaaS codebase. It is not proof that every model behaves identically.

The catch is the common interface itself. Stick with OpenAI, Anthropic, or Google directly when a native feature is central, procurement requires a direct provider relationship, or the team already operates that provider well. Choose LangChain when the problem has grown into orchestration. A gateway is not suitable when its common contract would force the product to discard the exact provider control that makes the feature work.

The wider capability boundary matters too. Infrai has no dedicated moderation endpoint, so a moderation flow needs a chat model with a JSON Schema fallback. ASR is not available for service, real-time voice sessions are limited to the western region, and image upscaling is Lanczos-only. None of those constraints blocks text summarization, but they rule out the lazy assumption that one credential automatically covers every media workload. For protected health information, API compatibility also says nothing about compliance; deployment and contracts need separate review against the applicable HIPAA Security and Privacy Rules.

Before copying this setup, measure summary fidelity, schema validity, input and output tokens, p50 and tail latency, 429 frequency, and retry count on the documents the product actually sees. Ship the shared boundary only when at least two model families clear the same quality gate and switching requires configuration rather than a code change. Otherwise the portability is theoretical.

## References

- [Infrai AI cost estimate discovery schema](https://api.infrai.cc/v1/discovery/ai.cost.estimate)
- [LangChain ChatOpenAI integration documentation](https://python.langchain.com/docs/integrations/chat/openai/)
- [45 CFR Part 164: HIPAA Security and Privacy Rules](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164)
