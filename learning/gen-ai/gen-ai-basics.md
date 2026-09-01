# Gen AI — Study Notes

## Reading list

- [x] Latent Diffusion
- [x] How Transformers are connected to LLMs, and their differences
- [x] Difference between AI, Machine Learning, and Deep Learning
- [x] Difference between grounding and tuning
- [ ] RAG pipeline
- [ ] Difference between ETL and RAG pipeline
- [ ] Prompt engineering
- [ ] Fine-tuning
- [ ] Indexing and semantic search
- [ ] Encoder-decoder pattern

## Contents

- [Transformer architecture & query processing](#transformer-architecture--query-processing)
- [Latent Diffusion Models (LDMs)](#latent-diffusion-models-ldms)
- [Vectors vs. tensors](#vectors-vs-tensors)
- [Transformers vs. LLMs](#transformers-vs-llms)
- [AI vs. ML vs. Deep Learning vs. Gen AI](#ai-vs-ml-vs-deep-learning-vs-gen-ai)
- [Grounding vs. fine-tuning](#grounding-vs-fine-tuning)
- [RAG pipeline](#rag-pipeline)

---

## Transformer architecture & query processing

Transformers use a **query-key-value** mechanism to process and understand relationships between words. The **self-attention** mechanism determines the importance of each word in a given text, and the final output is produced by the decoder.

### How a transformer responds to a query

1. The model starts with a blank token as the output.
2. It processes the input sentence and determines the most important word.
   - Example: for the input *"Where is New York?"*, the most important token is **New York**.
3. The decoder begins from that most important word.
4. It uses self-attention and encoder-decoder attention to combine this word with the input sentence.
5. The model predicts the next most relevant word based on its training.
6. The process repeats, joining each predicted word with the previous ones, until the full response is formed.

This is the **encoder-decoder architecture** that lets transformers generate meaningful responses.

---

## Latent Diffusion Models (LDMs)

A type of generative AI model used mainly for image work:

- Text-to-image conversion
- Image-to-image transformation
- Image generation

### How latent diffusion works

**1. Encoding**

- The input (text prompt) is converted into vector representations.
- The model captures the meaning of the words and their relationships.
- A **latent space** is created with a random noise tensor, which is the starting point for the image.

**2. Diffusion**

- The diffusion model gradually removes noise step by step.
- A neural network (**U-Net**) predicts how much noise to remove at each step.
- **Cross-attention** keeps the output aligned with the input text.
- Over multiple iterations, noise is progressively transformed into a structured image.

**3. Decoding**

- The decoder converts the latent-space representation into a high-resolution image.
- The result is a visually coherent, high-quality image.

### Example

For the prompt *"a cat sitting on a fence"*, the model:

1. Encodes the text into a vector representation.
2. Uses diffusion to generate the image, removing noise at each step.
3. Decodes the final high-resolution image of a cat sitting on a fence.

---

## Vectors vs. tensors

| Concept | Definition | Example | Shape |
| --- | --- | --- | --- |
| **Vector** | A 1D array — a list of numbers | `[2, 5, 7]` (3-element vector) | `(3,)` |
| **Tensor** | A multi-dimensional array; the generalized form of vectors and matrices | A 3D image representation (height, width, channels) | `(256, 256, 3)` |

- A vector is a 1D tensor.
- A tensor can be 1D, 2D, 3D, or higher-dimensional, and is used to represent complex data structures.
- Tensors are essential to deep learning because they can hold images, text embeddings, and other multi-dimensional numerical representations.

---

## Transformers vs. LLMs

Transformers are a deep learning **architecture** designed for processing and generating text. They are the core engine behind many AI models.

The relationship is like an engine and a car:

- A **Transformer** is the engine — the fundamental architecture that enables text processing.
- An **LLM** is the car — a fully developed model built on transformers and trained on massive datasets to understand and generate human-like text.

In short: LLMs use transformers, but add large-scale training data to improve language understanding and generation.

---

## AI vs. ML vs. Deep Learning vs. Gen AI

| Concept | Definition | Example |
| --- | --- | --- |
| **Artificial Intelligence (AI)** | The broadest field — making machines simulate human intelligence: reasoning, problem-solving, decision-making. | Virtual assistants like Siri and Alexa responding to voice commands. |
| **Machine Learning (ML)** | A subset of AI that lets machines learn patterns from data and improve without explicit programming. | Spam filters classifying emails based on past data. |
| **Deep Learning (DL)** | A subset of ML using neural networks over large datasets — speech recognition, image processing, NLP. | Self-driving cars detecting pedestrians and road signs. |
| **Generative AI (Gen AI)** | A subset of deep learning that generates new content (text, images, code) rather than just analyzing data. | ChatGPT generating text; DALL·E creating images from text. |

Nesting: **AI ⊃ ML ⊃ Deep Learning ⊃ Generative AI**

---

## Grounding vs. fine-tuning

**Grounding** provides a model with external data at *runtime* to help it generate more accurate responses — typically via an external vector database or lookup system. The common approach is **RAG (Retrieval-Augmented Generation)**, where relevant information is retrieved before the response is generated. Because it happens at runtime, grounding suits dynamic, frequently updated data.

**Fine-tuning** trains a foundation model on additional data so it adjusts its internal weights and permanently learns new information. It happens *before* the model is used and requires a dedicated training phase. It suits large, mostly static datasets, or cases where the model needs to learn a specific behavior — for example, building a code converter.

| Feature | Grounding | Fine-tuning |
| --- | --- | --- |
| **Definition** | Supplying external knowledge at runtime to improve accuracy | Adjusting the model's internal weights by training on new data |
| **How it works** | Retrieval-based — vector search (RAG) or API lookups fetch relevant data per query | A training process where the model learns from new examples |
| **Data storage** | Outside the model (e.g. Pinecone, Weaviate) | Inside the model, after training |
| **Flexibility** | Knowledge updates without retraining | Retraining required for new information |
| **Use case** | Knowledge that changes often — company policies, real-time prices, FAQs | Static, structured knowledge — e.g. a chatbot for airline policies |
| **Cost & time** | Lower; the model itself is untouched | Expensive and slow, especially for large models |
| **Example** | RAG fetches airline policy docs at query time | Model fine-tuned on examples like *"Economy passengers can carry 7kg luggage."* |

---

## RAG pipeline

<!-- TODO: not yet written up. Cover: ingestion → chunking → embedding → vector store →
     retrieval → re-ranking → prompt augmentation → generation.
     Then contrast with an ETL pipeline (next item on the reading list). -->

---

## Notes

<!-- Scratch space for questions and follow-ups. -->