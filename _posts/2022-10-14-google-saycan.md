---
layout:     post
title:      "Google SayCan"
subtitle:   ""
date:       2022-10-15 16:00:00
tags:
    - Tech
typora-root-url: ../
published: False
---

<!-- Recently, Google released an interesting project, SayCan, which demonstrates how to bridge the gap between what we want robots to do (in natural language) and what robots can do (atomic functionalities).
While the video demo is quite impressive, the paper and the code seem overwhelming at the first glance.
In this post, I will try to summarize the key ideas of the paper and the code, and provide a simplified implementation of SayCan. -->

Nowadays, large language models such as ChatGPT have demonstrating their ability to generate natural talks.
This gives rise to a fascinating possibility: can we use such language models to guide robots to perform tasks?
For example, let's ask ChatGPT to "help me clean up my spilled drink":
```
As a language model AI, I do not have the physical ability to clean up spills. But here are some steps you can follow to clean up a spilled drink:  
1. Blot the spill with a clean cloth or paper towel. Do not rub or scrub the spill, as this can spread it further.  
2. Use a household cleaner or dish soap mixed with water to clean the spill.  
3. Rinse the area with water and blot again with a clean cloth or paper towel.  
4. Repeat the process until the spill is completely cleaned up.  
5. Allow the area to air dry or use a fan to speed up the drying process.  
Note: If the spill is on a carpet or fabric, be sure to test the cleaner on a small, inconspicuous area first to make sure it won't cause damage.
```

We can see two essential problems in ChatGPT's response:
1. Robots often implement limited capabilities (or skills for short), but the language model is not aware of what the robot can do, and cannot convert the instructions into specific robot functionalities.
2. The language model is not aware of the real-world, thus it gives instructions that are vague (e.g., `with a clean cloth or paper towel`) or not practical (if we don't have a soap at hand).

The key observation of this paper is to score the likelihood among all the possible skills the robot can perform, and form a sequence with maximum likelihood.
More specifically, at each step, the score of a skill should be proportional to 1. how much the skill is **logically** related to the goal, and 2. how much the skill is **practically** doable.
We can formally describe how to select the skill $\pi$ for a given instruction $i$ with the following equation:
$$
argmax_{\pi \in \Pi} p(c_\pi|s,l_\pi)p(l_\pi|i)
$$
where $l_\pi$ denotes the description of the skill, the first part denotes the possibility for **practically** completing the skill in the current situation, and the second part denotes the possibility for the skill to be **logically** related to the instruction $i$.

Now let's see how to implement this idea, with a close look into the required knowledge and the code (it released [a giant ipynb](https://github.com/google-research/google-research/blob/master/saycan/SayCan-Robot-Pick-Place.ipynb) which is scary at first glance).

# Language Models

Whatever the underlying implementation, you can think of a language model as a function that maps a sequence of words to a probability distribution for the next word.
Formally, a language model computes $p(w_{j}|w_{1},\dots,w_{j-1})$.
For example, if you feed the sentence "A dog is an" to a language model, it would give you the following probability distribution for the next word:
```
animal: 0.5
pet: 0.2
friend: 0.1
...
```
To assess the possibility of a piece of text denoted as $W={w_0, w_1, \dots, w_n}$, we can compute with the chain rule:
$$
p(W) = \prod_{i=0}^n p(w_i|w_0, \dots, w_{i-1})
$$

And, if a prefix is given, say $w_0, w_1, \dots, w_{j-1}$, the possibility of the whole sentence can be computed as:
$$
p(W) = \prod_{i=j}^n p(w_i|w_0, \dots, w_{i-1})
$$

Now you may have got the idea, let's see how the paper implemented this.
```python
def gpt3_scoring(query, options, engine="text-ada-001", limit_num_options=None, option_start="\n", verbose=False, print_tokens=False):
  if limit_num_options:
    options = options[:limit_num_options]
  verbose and print("Scoring", len(options), "options")
  gpt3_prompt_options = [query + option for option in options]
  response = gpt3_call(
      engine=engine, 
      prompt=gpt3_prompt_options, 
      max_tokens=0,
      logprobs=1, 
      temperature=0,
      echo=True,)
  
  scores = {}
  for option, choice in zip(options, response["choices"]):
    tokens = choice["logprobs"]["tokens"]
    token_logprobs = choice["logprobs"]["token_logprobs"]

    total_logprob = 0
    for token, token_logprob in zip(reversed(tokens), reversed(token_logprobs)):
      print_tokens and print(token, token_logprob)
      if option_start is None and not token in option:
        break
      if token == option_start:
        break
      total_logprob += token_logprob
    scores[option] = total_logprob

  for i, option in enumerate(sorted(scores.items(), key=lambda x : -x[1])):
    verbose and print(option[1], "\t", option[0])
    if i >= 10:
      break

  return scores, response
```

# Remaining Engineering Challenges

Skills granularity.
