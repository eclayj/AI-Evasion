# Fundamentals of AI — Module 290

These are original study notes for the HTB Academy AI Red Teamer path. They follow the course one section at a time and explain security connections for a learner comfortable with programming but new to AI security. A course concept is identified as such; a security implication or engineering caveat is labeled separately. No live target, token, or answer belongs in this file.

## Section 1 of 24 — Introduction to Machine Learning

### The three nested terms

| Term | Plain-English meaning | Example |
| --- | --- | --- |
| Artificial intelligence (AI) | The broad effort to make a system perform tasks associated with intelligence, such as interpreting language, recognizing objects, reasoning, or making decisions. | A rule-based troubleshooting expert system or an image-recognition service. |
| Machine learning (ML) | A way to build AI behavior by fitting a model to examples, rather than writing every decision rule by hand. | A spam filter trained on labeled messages. |
| Deep learning (DL) | A family of ML methods that uses neural networks with multiple layers to learn useful representations from data. | A convolutional network that learns visual patterns in images. |

Read the relationship as **deep learning is inside machine learning, and machine learning is inside AI**. The words are not interchangeable. A hand-written rule engine may be AI without being ML. A decision tree is ML without being deep learning. A neural network is not automatically *deep* just because it has the word “network” in its name; depth refers to multiple learned layers. The nested-circle picture in HTB is a teaching aid, not a claim that every intelligent behavior fits one fixed taxonomy.

For a data-engineering analogy, think of AI as the overall product capability (“classify suspicious events”), ML as a component that learns a mapping from historical records, and DL as one possible model family for that component. A production system may also contain non-ML rules, databases, APIs, human review, and monitoring. The model is not the whole application.

### Three ways a learning problem can provide feedback

1. **Supervised learning:** each training example includes a desired answer, or *label*. For email filtering, an example is a message plus “spam” or “not spam.” Training adjusts a model so its predictions better match those labels. Classification predicts a category; regression predicts a number. HTB introduces the category here and teaches individual algorithms later.
2. **Unsupervised learning:** examples arrive without those answer labels. The model looks for structure such as groups, unusual points, or lower-dimensional representations. A cluster of similar network events is a pattern, not automatically an attack. A human or a separate process must interpret what the pattern means.
3. **Reinforcement learning:** an *agent* chooses an action, the *environment* changes, and the agent receives a reward or penalty. Over many interactions it learns a *policy*—a rule for choosing actions in situations. This differs from being handed the correct action for every example. A game-playing agent learning from wins and losses is the course's intuitive example.

The easiest question to ask is: **Where does the feedback come from?** A supplied answer means supervised learning; no supplied answer means unsupervised learning; rewards after actions mean reinforcement learning. Real systems may combine these approaches, but keeping the basic cases separate will help when we reach their algorithms.

### What deep learning adds

Traditional ML often depends on people choosing useful input features—for instance, message length, sender reputation, and the presence of suspicious phrases. A deep network can learn intermediate features from rawer inputs. In an image system, early layers may react to edges or textures; later layers combine those responses into more complex shapes. “Early” and “later” describe processing stages, not a guarantee that every neuron has a neat human-readable meaning.

HTB names convolutional neural networks (CNNs) for images, recurrent neural networks (RNNs) for sequences, and transformers for language. Treat these as architecture families optimized for different data relationships, not as rules that restrict each family to only one task. “End-to-end” training means the final task's learning signal can adjust multiple stages together; it does not mean data collection, validation, deployment, and security happen automatically.

### Cybersecurity bridge (additional context)

- A **false negative** is a real threat that a detector misses. A **false positive** is benign activity that it flags. These errors have different costs: a missed intrusion can be damaging, while too many false alerts can overwhelm analysts.
- A model can learn only from the examples and labels it receives. An attacker who changes training records or labels may contaminate what it learns; that is a preview of **data poisoning**. An attacker who changes an input at prediction time to evade a trained detector is attempting **evasion**. Later path modules examine these attacks directly.
- Unsupervised anomaly detection asks “does this differ from the usual pattern?” It does **not** answer “is this malicious?” A new legitimate deployment can look anomalous, and a careful attacker can look ordinary.
- Reinforcement learning optimizes the stated reward. If the reward is a poor stand-in for the real safety objective, an agent may find a high-scoring but unwanted behavior. Security review must examine the reward definition and what the agent can control.

### Check your understanding

- A fixed list of hand-written fraud rules: potentially AI, but not ML just because it makes decisions.
- A decision tree trained on labeled fraud cases: ML and supervised learning, but not DL.
- Grouping unlabeled traffic records by similarity: unsupervised ML; the groups still need interpretation.
- A multi-layer neural network trained to identify objects in labeled images: supervised DL (and therefore also ML and AI).

There is no new mathematical notation or course code in this section. We will add equations and annotated notebook cells when the course introduces algorithms that need them, rather than pre-implementing later sections here.
