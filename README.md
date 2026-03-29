# sesotho-llm-prompting
Prompting LLMs in Sesotho: Findings from a Production System

Author: Lebohang Kuenane — @kuenane
Context: PayPesa — Lesotho's first PayPal-to-M-Pesa remittance app
Model tested: claude-3-opus-20240229
Task: Generating SMS notifications in Sesotho for remittance recipients
Status: Ongoing — findings updated as the system evolves
Why This Exists
Sesotho (Sesotho sa boroa/ Southern Sotho) is a Bantu language spoken by approximately 5–6 million people, primarily in Lesotho and South Africa. It is severely underrepresented in the training corpora of large language models.
When I integrated Claude into PayPesa to generate recipient SMS notifications, I assumed language control would be straightforward — specify the language, get the output. It was not. What followed was months of iterative prompt engineering against a model that had clearly seen far less Sesotho than English, and the failures were not random. They were patterned, reproducible, and instructive.
This document records what I found: the failure modes, the prompt architectures I tested, what worked, what didn't, and the open questions I haven't resolved yet.
This is not a benchmark study. I had no labelled ground-truth dataset, no native-speaker annotation pipeline, and no compute budget. What I had was a production system, real users, and my own native fluency in Sesotho as the evaluation instrument.
The Task
Each time a PayPesa transfer completes (or is queued), the system generates an SMS notification for the recipient. Requirements:
Written in Sesotho (not English, not a mix)
160 characters or fewer (standard SMS limit)
Correct recipient name, amount in LSL (Lesotho Loti), and sender reference
Appropriate register: warm but professional, not robotic
No hallucinated amounts — the loti figure must match the input exactly
A typical target output looks like:
Lumela [Name]! O amohetse LSL [amount] ho tsoa ho [sender]. 
Lefa e fihlile. - PayPesa
Failure Modes Observed
1. Wrong Grammar and Verb Conjugation
Sesotho has a complex noun class system (similar to other Bantu languages) that governs verb agreement, pronoun use, and adjective forms. Claude's grammatical failures were consistent in character:
Subject-verb agreement errors — using the wrong concord prefix for the noun class
Tense markers applied incorrectly — particularly with the perfect aspect
Possessive constructions reversed or anglicised
Example of conjugation error (reconstructed):
Output (incorrect)
Correct form
Issue
O romela chelete
O rometse chelete
Wrong tense — present instead of perfect
Chelete e teng
Chelete e fihlile
Wrong verb for "has arrived" in this context
These errors were not random — they recurred in the same grammatical positions, suggesting the model had learned some Sesotho surface patterns but not the underlying morphological rules.
2. Misspelled Sesotho Words
Sesotho orthography includes characters and diacritical patterns that differ from English. Claude produced systematic misspellings:
Dropping the h in digraphs like tlh, tsh, hlh
Substituting e for ê and o for ô where tone or vowel quality matters
Anglicising loanwords that have established Sesotho spelling conventions
These are not typos — they reflect gaps in training data coverage of written Sesotho, particularly formal/standardised orthography.
3. Language Mixing (Code-Switching Without Instruction)
The most common failure mode: Claude would begin in Sesotho and drift into English within the same sentence or paragraph, without any instruction to do so.
This is distinct from intentional code-switching (which is natural in spoken Sesotho). The model was not mirroring natural usage — it was defaulting to English when it ran out of confident Sesotho representations for a concept.
Trigger patterns I observed:
Financial terminology — "balance", "transaction", "transfer" triggered English fallback even where Sesotho equivalents exist
Numbers and currency — amounts were frequently rendered in English syntax even when surrounding text was Sesotho
Politeness formulas — closing phrases defaulted to English ("Thank you" instead of "Kea leboha")
Prompt Architectures Tested
Experiment 1: System Prompt vs. User Turn for Language Instruction
Hypothesis: Where you place the language instruction affects output stability.
Setup:
Variant A: Language instruction in the system prompt only
Variant B: Language instruction in the user turn only
Variant C: Language instruction in both
Finding:
Placing the language instruction exclusively in the system prompt produced the most consistent Sesotho output. User-turn-only instruction was the least stable — Claude treated it more like a preference than a constraint, and language bleed occurred more frequently.
Variant C (both) was marginally better than system-prompt-only, but the improvement was small. The system prompt placement appears to be the primary anchor.
Recommended pattern:
const systemPrompt = `
You are a Sesotho language SMS generator for a financial application.
CRITICAL: All output must be written entirely in Sesotho (Southern Sotho / Sesotho sa Leboa).
Do not use English. Do not mix languages. If you are uncertain of a Sesotho term, 
use the closest correct Sesotho equivalent — never substitute English.
`;
Experiment 2: Explicit Character Limits
Hypothesis: Explicit character constraints affect output length and structure.
Setup:
Variant A: No character limit specified
Variant B: "Keep the SMS under 160 characters"
Variant C: "Your output must be 160 characters or fewer. Count carefully before responding."
Finding:
Variant A produced outputs frequently exceeding 300+ characters — verbose, paragraph-style notifications that would fail SMS delivery.
Variant B improved this significantly but was unreliable — Claude would occasionally overshoot, seemingly not counting.
Variant C ("count carefully before responding") was the most reliable, but still not perfect. Adding a character count request in the output format — "Reply with the SMS text only, followed by: [CHARS: n]" — forced the model to self-monitor length and reduced overruns meaningfully.
Recommended pattern:
const userPrompt = `
Generate a Sesotho SMS notification for the following transfer:
- Recipient: ${name}
- Amount: LSL ${amount}
- Sender: ${sender}

Requirements:
- Entirely in Sesotho
- 160 characters or fewer (count carefully)
- Reply with the SMS text only, then on a new line: [CHARS: n]
`;
The [CHARS: n] tag is then parsed server-side and used to flag and retry any generation exceeding the limit.
Experiment 3: "Think in Sesotho First"
Hypothesis: Instructing Claude to reason in Sesotho before generating output would reduce language bleed.
Setup:
Added to the prompt: "Before writing the SMS, think through the key phrases you will use in Sesotho. Then write the final SMS."
Finding:
Mixed results. On positive cases, this produced better Sesotho — the intermediate reasoning step seemed to "commit" the model to the language before generation. On negative cases, the reasoning step itself was in English, which then bled into the output.
A modified version — "Write three possible Sesotho SMS drafts, then select the best one" — was more consistently effective. The multi-draft approach reduced single-output variance and the selected output was almost always the strongest of the three.
Tradeoff: ~3x token cost. Acceptable for a low-volume production system; would need optimisation at scale.
What I Have Not Solved
These are open problems I am still working on:
1. No automated evaluation. My quality assessment is entirely manual — I read the output and judge it as a native speaker. This does not scale, and it is not reproducible. I am working toward a lightweight eval harness, but building one without a labelled Sesotho dataset is a research problem in itself.
2. Financial terminology gaps. There is no consensus Sesotho vocabulary for many fintech concepts ("remittance", "exchange rate", "transfer fee"). Claude's handling of these is inconsistent. I have experimented with providing a glossary in the system prompt; results are promising but not yet stable.
3. Tone calibration. "Warm but professional" is easy to specify in English. It is harder to define in a Sesotho prompt in a way that Claude reliably executes. Current outputs range from cold/transactional to overly informal.
4. Model version sensitivity. All findings here are from claude-3-opus-20240229. I have not systematically tested other model versions. Sesotho capability likely varies significantly across versions and I do not yet know the direction of that variation.
Broader Observations
Low-resource language prompting is a different problem than translation.
Claude is not bad at translating English into Sesotho when asked explicitly. But generation-from-scratch in Sesotho — where the model must compose rather than convert — surfaces much more instability. The translation pathway seems to activate different (and better-calibrated) internal representations.
Failure modes are morphologically structured, not random.
Claude's Sesotho errors follow grammatical patterns. This suggests the model has partial linguistic knowledge — it has seen enough Sesotho to learn surface regularities but not enough to internalise the morphological rule system. This is a more tractable problem than zero-shot ignorance: it implies targeted fine-tuning or few-shot exemplar injection could be effective.
Prompt engineering is not a substitute for training data.
Every technique here is a compensation mechanism. The underlying problem is that Sesotho is underrepresented in pretraining. Prompt engineering can improve output quality significantly, but it cannot close the gap that better training coverage would close. This is worth naming clearly.
Next Steps
[ ] Build a structured eval harness using a hand-labelled set of 50–100 Sesotho SMS examples (native speaker annotation)
[ ] Test the multi-draft selection approach systematically against single-shot generation
[ ] Compile a financial terminology glossary in Sesotho and evaluate system-prompt injection vs. RAG retrieval
[ ] Test newer Claude model versions against the same prompt suite
[ ] Publish labelled eval dataset for community use
Contributing
If you are a Sesotho speaker, a researcher working on Bantu language NLP, or someone building LLM systems for low-resource African languages — I would genuinely like to hear from you.
Open an issue, or reach out directly: lebohang.kuenane44@gmail.com
About PayPesa
PayPesa is Lesotho's first PayPal-to-M-Pesa remittance application, serving the financial corridor between the diaspora and 500,000+ Basotho with no existing digital solution. Built and maintained by a single engineer, without funding or a team.
github.com/kuenane
