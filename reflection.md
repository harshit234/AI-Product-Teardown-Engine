# Reflection

## 1. Which of the 6 layers surprised you the most in terms of complexity for the product you chose? Why?

Layer 6 — System Design & Scale surprised me the most. Initially, I assumed the main challenge would be building accurate machine learning models, but I realized that serving real-time personalized recommendations to millions of users within strict latency constraints is far more complex. The system must balance accuracy and speed using techniques like multi-stage ranking and approximate nearest neighbor search, which introduces trade-offs that are not obvious at first. Additionally, handling real-time signals like location and availability while maintaining system reliability makes this layer highly challenging.

## 2. What was the single biggest difference you noticed between the LLMs you tested? Not just "one was better" — what specifically did one do that the others couldn't?

The biggest difference I observed was in how different LLMs handled specificity and reasoning depth. One model (e.g., Claude/DeepSeek) provided deeper insights into engineering challenges and trade-offs, such as latency vs accuracy in recommendation systems, which made the analysis more realistic. In contrast, ChatGPT was more consistent in following the required structure and avoided hallucinations by using cautious language like "likely," making it more reliable for production use. This showed that some models are better at reasoning, while others are better at structured, dependable outputs, which is important when choosing an LLM for real-world applications.
