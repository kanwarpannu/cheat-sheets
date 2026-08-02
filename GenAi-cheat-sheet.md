# GEN AI - Generative AI Introduction

## LLM  
### Basic Architecture  
Taking the example of a Llama 2 70B model, an LLM will contain 2 files: a parameter file and a run file.

A **Parameter** file (of the neural network used)
will be 140GB for a 70B model because it uses the float16 datatype which takes 2 bytes per parameter,
thus 2 × 70B = 140GB.

This file is a byproduct of all the training done.
It can be seen as a compressed version of multiple TB of training data along with various weights
given to each data piece — in general it is roughly 100 times smaller than its training data.

Essentially, the neural network is trained to learn about world data so that it can predict the next word.  

The **Run** file is used to run the neural network model.
This process is also called **Inference**.
It can be written in any language — in C, it is roughly 500 lines of code.  

Since the 70B model has 70 billion parameters,
it is practically impossible to know exactly how the model arrives at any given output.
However, we can tweak these parameters and their weights during inference
to make it better at next word prediction.  

### Training Stages  
1. The process of getting an initial model created after training on a dataset (~10TB of internet data compressed into a neural network) is called **Pre-training**. This is very computationally intensive and needs a cluster of thousands of GPUs. The end product is referred to as the **Base Model**.  
2. The Base Model is then trained on ~100K high-quality Q&A-style responses in a step called **Supervised Fine-Tuning (SFT)**, producing an **Assistant Model**. This step can be run multiple times to iteratively improve the model. It was initially done primarily by humans, but AI-generated responses are increasingly used alongside human feedback.  
3. An optional third stage called **Reinforcement Learning from Human Feedback (RLHF)** further aligns the model to human preferences — improving helpfulness, safety, and output quality. Multiple answers are generated, ranked by humans (or another model), and the best are used to update the model.  

---

## Common Terminology
1. **Models:**  
They go through large datasets in a process called **Training** and find patterns to make predictions accordingly.
   They **DO NOT** store that information, so they cannot work as databases.
   They do not understand the results they are generating,
   so they won't know fact from fiction or recognize a pattern they have not been trained on.
   Therefore, LLMs are essentially next word predictors.  
2. **LLM vs Foundational Models:**  
An AI model trained on a text-based dataset is called an LLM.
However,
   all models are also called foundational models (LLM is a subpart of it),
   which is a word used to describe other AI models like image,
   video and audio etc.
3. **Multimodal AI:**  
Multimodal AI are models that can process or generate more than one type of data e.g., text and image at the same time.  
4. **Parameters:**  
Numerical values (weights and biases) inside a neural network that are adjusted during training.
   More parameters generally means a more capable but resource-intensive model
   e.g., GPT-3 had 175 billion parameters, Llama had 65 billion.  
5. **Token:**  
The smallest unit of text a model processes. Text is split into tokens before training or inference
   (roughly 1 token ≈ ¾ of a word in English).
   Training data is measured in tokens — GPT-3 was trained on 300 billion tokens, Llama on 1.4 trillion.  
6. **Quantization:**  
The process of mapping 32-bit float parameters to use smaller values like 8-bit integers.
   This reduces precision of weight values, which can slightly lower accuracy.
   However, the trade-off is significantly less storage and faster inference (faster computation).
   A well-balanced quantized model can retain most accuracy while being substantially smaller and faster.  
7. **Hallucination:**  
When a model generates confident-sounding but factually incorrect or fabricated output.
   This happens because models predict statistically likely text, not verified facts.
   Always verify LLM output for any factual, legal, or safety-critical information.  
8. **Temperature:**  
A parameter set at inference time that controls how random or creative the model's output is.
   Low temperature (near 0) → more deterministic and predictable responses.
   High temperature (near 1 or above) → more varied and creative but potentially less coherent.  
9. **Embeddings:**  
A way of representing text (words, sentences, documents) as numerical vectors in high-dimensional space.
   Similar meanings end up close together in that space, enabling semantic search and powering systems like RAG.
   Embedding models are separate from generative models and are used as a preprocessing step.  
10. **Context Window:**  
The maximum number of tokens a model can "see" at once — both input and output combined.
   Text outside this window is not visible to the model, which is why very long conversations can cause the model to forget earlier content.
   Larger context windows allow processing of longer documents but require more memory and compute.  
11. **ReAct (Reasoning + Acting):**  
A pattern for AI agents that alternates between thinking and doing to solve multi-step problems.
   The agent loops through: **Thought** (reason about what to do next) → **Action** (call a tool or take a step) → **Observation** (see the result) — repeating until a final answer is reached.
   The intermediate Thought/Action/Observation steps form an ephemeral scratchpad that helps the model build on its own reasoning, then is discarded once the response is returned.  

---

## Prompt Engineering
The practice of crafting inputs to guide a model toward better outputs.

- **System prompt:** Instructions given to the model before the conversation begins, setting its role, tone, or constraints.
- **User prompt:** The actual input/question from the user during a session.
- **Zero-shot:** Asking the model to complete a task with no examples — just the instruction.
- **Few-shot:** Providing a few input/output examples in the prompt so the model learns the pattern before answering.
- **Chain-of-thought:** Asking the model to "think step by step" before giving a final answer, which improves accuracy on reasoning tasks.

---

## RAG (Retrieval-Augmented Generation)
A pattern where relevant documents are fetched from an external knowledge base and injected into the prompt before the model generates a response.
Pairs naturally with embeddings — documents are stored as vectors and retrieved by semantic similarity to the query.

**Why it matters:** Models have a training cutoff and cannot access private data. RAG gives them up-to-date or domain-specific context without retraining.

Flow: `User query → Embed query → Search vector DB → Inject top results into prompt → Generate response`

---

## Memory (in AI Agents)
Agents need memory to reason across steps, sessions, and knowledge bases. Without it, every call starts from scratch — the agent has no awareness of what it did before or what it already knows.

**1. Ephemeral Memory**  
Exists only during a single step or tool call, then is discarded immediately.  
- **Storage:** In-process variables (RAM only) — no persistence of any kind.  
- **Use case:** A ReAct-style agent's scratchpad — intermediate reasoning steps and tool outputs held while composing a final answer. Once the response is returned, this memory is gone.  

**2. Short-Term Memory**  
Lives for the duration of a conversation session.  
- **Storage:** In-memory list of messages (conversation history) passed in the context window each turn. Some frameworks also store it in a session cache (e.g., Redis) tied to a session ID.  
- **Use case:** Multi-turn chat where the model refers back to what was said earlier in the same conversation — e.g., "as I mentioned above…". The context window is the hard limit on how much short-term memory exists.  

**3. Long-Term Memory**  
Persists across sessions — survives when the conversation ends and is available in future ones.  
- **Storage:** Vector databases (Pinecone, pgvector, Chroma) for semantic retrieval; relational databases (Postgres, SQLite) for structured facts; plain files for simple key-value recall. Retrieved into context via RAG.  
- **Use case:** A personal assistant agent that remembers your name, preferences, and past decisions. A support bot that recalls previous tickets. An autonomous agent that logs completed steps to avoid repeating work.  

**Agent Memory Flow:**  
`User input → Read long-term memory (RAG from vector DB) → Load short-term memory (conversation history) → Process with ephemeral scratchpad (reasoning steps) → Respond → Write important outcomes back to long-term memory`
