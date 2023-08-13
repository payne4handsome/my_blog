---
title: "Openai GPT Prompt 官方教程"
date: 2023-07-31T14:02:00+08:00
draft: False
categories: []
tags: [prompt, gpt, openai, chatgpt, gpt-4]
author: pan
---

## openai官方教程(六大策略)

Six strategies for getting better results

### 一、Write clear instructions

#### Include details in your query to get more relevant answers

在你的问题中包含细节，以获得更相关的答案

| bad   |      good      |
|----------|-------------|
| Who’s president? |  Who was the president of Mexico in 2021, and how frequently are elections held? |
| Write code to calculate the Fibonacci sequence. |    Write a TypeScript function to efficiently calculate the Fibonacci sequence. Comment the code liberally to explain what each piece does and why it's written that way.   |
|  Summarize the meeting notes.| Summarize the meeting notes in a single paragraph. Then write a markdown list of the speakers and each of their key points. Finally, list the next steps or action items suggested by the speakers, if any. |

#### Ask the model to adopt a persona

prompt中指定角色（openai api中通过角色system来设定角色）
| role   |content |
|----------|----------|
|SYSTEM|When I ask for help to write something, you will reply with a document that contains at least one joke or playful comment in every paragraph.|
|USER|Write a thank you note to my steel bolt vendor for getting the delivery in on time and in short notice. This made it possible for us to deliver an important order.|

#### Use delimiters to clearly indicate distinct parts of the input

用分割符清楚的定义需要语言模型处理的部分。

+ 例1

```text
Summarize the text delimited by triple quotes with a haiku.

"""insert text here"""
```

+ 例2

| role   |content |
|----------|----------|
|SYSTEM|You will be provided with a pair of articles (delimited with XML tags) about the same topic. First summarize the arguments of each article. Then indicate which of them makes a better argument and explain why.|
|USER|\<article\> insert first article here \</article\> <br> \<article\> insert second article here \</article\>|

#### Specify the steps required to complete a task

| role   |content |
|----------|----------|
|SYSTEM|Use the following step-by-step instructions to respond to user inputs.<br><br> Step 1 - The user will provide you with text in triple quotes. Summarize this text in one sentence with a prefix that says "Summary: ". <br><br> Step 2 - Translate the summary from Step 1 into Spanish, with a prefix that says "Translation: ".|
|USER|"""insert text here"""|

#### Provide examples(Few-shot)

提供例子比详细的描述任务要更有效，而且有的时候把任务描述清楚是比较空难的。**所以推荐把更多的时间花在打磨示例上**

| role   |content |
|----------|----------|
|SYSTEM|Answer in a consistent style.|
|USER|Teach me about patience.|
|ASSISTANT|The river that carves the deepest valley flows from a modest spring; the grandest symphony originates from a single note; the most intricate tapestry begins with a solitary thread.|
|USER|Teach me about the ocean.|

#### Specify the desired length of the output

指定希望输出的长度，可以指定输出单词的数量、句子的数量、段落的数量、核心观点的数量。**但是指定输出单词的数量，语言模型可能不会那么精准。一般而言指定输出的段落或者观点的数量要更可靠一点**

| role   |content |
|----------|----------|
|USER|Summarize the text delimited by triple quotes in about 50 words.<br><br>"""insert text here"""|

| role   |content |
|----------|----------|
|USER|Summarize the text delimited by triple quotes in 2 paragraphs.<br><br>"""insert text here"""|


| role   |content |
|----------|----------|
|USER|Summarize the text delimited by triple quotes in 3 bullet points.<br><br>"""insert text here"""|

### 二、Provide reference text

GPT可以确信的编造假的回答，尤其当问专业内容或者引用和URL时候。提供参考可以减少这个问题

#### Instruct the model to answer using a reference text

指示模型使用提供的信息回答问题

| role   |content |
|----------|----------|
|SYSTEM|Use the provided articles delimited by triple quotes to answer questions. If the answer cannot be found in the articles, write "I could not find an answer."|
|USER|\<insert articles, each delimited by triple quotes\><br><br>\<insert articles, each delimited by triple quotes\> <br>Question: \<insert question here\>|

#### Instruct the model to answer with citations from a reference text

| role   |content |
|----------|----------|
|SYSTEM|You will be provided with a document delimited by triple quotes and a question. Your task is to answer the question using only the provided document and to cite the passage(s) of the document used to answer the question. If the document does not contain the information needed to answer this question then simply write: "Insufficient information." If an answer to the question is provided, it must be annotated with a citation. Use the following format for to cite relevant passages ({"citation": …}).|
|USER|\<insert document here\> <br>Question: \<insert question here\>|


示例

```text
# system
You will be provided with a document delimited by triple quotes and a question. Your task is to answer the question using only the provided document and to cite the passage(s) of the document used to answer the question. If the document does not contain the information needed to answer this question then simply write: "Insufficient information." If an answer to the question is provided, it must be annotated with a citation. Use the following format for to cite relevant passages ({"citation": …}).

# user

"""A steam engine is a heat engine that performs mechanical work using steam as its working fluid. The steam engine uses the force produced by steam pressure to push a piston back and forth inside a cylinder. This pushing force can be transformed, by a connecting rod and crank, into rotational force for work. The term "steam engine" is generally applied only to reciprocating engines as just described, not to the steam turbine. Steam engines are external combustion engines, where the working fluid is separated from the combustion products. The ideal thermodynamic cycle used to analyze this process is called the Rankine cycle. In general usage, the term steam engine can refer to either complete steam plants (including boilers etc.), such as railway steam locomotives and portable engines, or may refer to the piston or turbine machinery alone, as in the beam engine and stationary steam engine.

Although steam-driven devices were known as early as the aeolipile in the first century AD, with a few other uses recorded in the 16th century, in 1606 Jerónimo de Ayanz y Beaumont patented his invention of the first steam-powered water pump for draining mines. Thomas Savery is considered the inventor of the first commercially used steam powered device, a steam pump that used steam pressure operating directly on the water. The first commercially successful engine that could transmit continuous power to a machine was developed in 1712 by Thomas Newcomen. James Watt made a critical improvement in 1764, by removing spent steam to a separate vessel for condensation, greatly improving the amount of work obtained per unit of fuel consumed. By the 19th century, stationary steam engines powered the factories of the Industrial Revolution. Steam engines replaced sails for ships on paddle steamers, and steam locomotives operated on the railways.

Reciprocating piston type steam engines were the dominant source of power until the early 20th century, when advances in the design of electric motors and internal combustion engines resulted in the gradual replacement of steam engines in commercial usage. Steam turbines replaced reciprocating engines in power generation, due to lower cost, higher operating speed, and higher efficiency. 

-Excerpted from https://en.wikipedia.org/wiki/Steam_engine"""

Question: What was the first commercially successful steam engine?

```

回答如下：

```text
The first commercially successful steam engine was developed in 1712 by Thomas Newcomen. James Watt made a critical improvement to the steam engine in 1764. 
{"citation": "[...] the first commercially successful engine that could transmit continuous power to a machine was developed in 1712 by Thomas Newcomen. 
James Watt made a critical improvement in 1764, by removing spent steam to a separate vessel for condensation, greatly improving the amount of work obtained per unit of fuel consumed."}
```

### 三、Split complex tasks into simpler subtasks

将复杂任务划分为简单任务

#### Use intent classification to identify the most relevant instructions for a user query

对用户的意图进行分类，匹配最相关的指令。

``` text
# system
You will be provided with customer service queries. Classify each query into a primary category and a secondary category. Provide your output in json format with the keys: primary and secondary.

Primary categories: Billing, Technical Support, Account Management, or General Inquiry.

Billing secondary categories:
- Unsubscribe or upgrade
- Add a payment method
- Explanation for charge
- Dispute a charge

Technical Support secondary categories:
- Troubleshooting
- Device compatibility
- Software updates

Account Management secondary categories:
- Password reset
- Update personal information
- Close account
- Account security

General Inquiry secondary categories:
- Product information
- Pricing
- Feedback
- Speak to a human

# USER

I need to get my internet working again.

```

#### For dialogue applications that require very long conversations, summarize or filter previous dialogue

由于GPT的输入长度是有限制的，对于对话应用来讲，输入的长度不可能一直的累计。所以解决这个问题需要一些变通的办法。（1）一旦达到输入长度限制，总结之前的会话，把总结后的文本包含在新的输入中。（2）过滤之前的内容，匹配与当前问题最相关的内容。比如可以通过embedding匹配的方式

#### Summarize long documents piecewise and construct a full summary recursively

当需要总结的文本长度长度加上输出的总结长度超过GPT限制的时候。GPT是无法响应的。解决办法为先分段总结然后再一起总结。

### 四、Give GPTs time to "think"

当要GPT立即给出答案时候，GPT可能会犯错误。在给出答案之前，让GPT给出推理步骤（chain of thought），可以帮助GPT给出更可靠的答案

#### Instruct the model to work out its own solution before rushing to a conclusion

在让模型匆忙给出答案之前，让模型给出自已的解决问题的步骤。

#### Use inner monologue or a sequence of queries to hide the model's reasoning process

在某些应用中，不到最后，不可以给出最后的答案。此种情况可以用内心独白或者查询序列去隐藏推理过程。

```text
SYSTEM
Follow these steps to answer the user queries.

Step 1 - First work out your own solution to the problem. Don't rely on the student's solution since it may be incorrect. Enclose all your work for this step within triple quotes (""").

Step 2 - Compare your solution to the student's solution and evaluate if the student's solution is correct or not. Enclose all your work for this step within triple quotes (""").

Step 3 - If the student made a mistake, determine what hint you could give the student without giving away the answer. Enclose all your work for this step within triple quotes (""").

Step 4 - If the student made a mistake, provide the hint from the previous step to the student (outside of triple quotes). Instead of writing "Step 4 - ..." write "Hint:".
USER
Problem Statement: <insert problem statement>

Student Solution: <insert student solution>
```

#### Ask the model if it missed anything on previous passes

询问模型在上一个回答中是否遗漏了什么

### 五、Use external tools

使用其它工具，比如问GPT一部电影的信息，可以通过外部工具把电影的演员、导演等信息召回（embedding召回），然后送入GPT。

#### Use embeddings-based search to implement efficient knowledge retrieval

第一步，通过embedding召回query的相关信息，第二步，将召回的相关信息送入GPT

#### Use code execution to perform more accurate calculations or call external APIs

GPT在算术计算时候很有可能会不正确，所以这里更好的做法是让GPT先生成代码，然后执行该段代码获取结果。

``` text

# SYSTEM
You can write and execute Python code by enclosing it in triple backticks, e.g. ```code goes here```. Use this to perform calculations.

# USER
Find all real-valued roots of the following polynomial: 3*x**5 - 5*x**4 - 3*x**3 - 7*x - 10.

```

GPT 返回


To find the real-valued roots of the polynomial, we can use the numpy library in Python. Here's how you can do it:

```python
import numpy as np

# Define the polynomial
coefficients = [3, -5, -3, -7, -10]

# Find the roots
roots = np.roots(coefficients)

# Filter out the complex roots
real_roots = [root for root in roots if np.isreal(root)]

# Print the real roots
for root in real_roots:
    print(root)
```

When you run this code, you will get the following real-valued roots:

```shell
-1.0
-1.0
-0.3333333333333333
2.0
```

#### Give the model access to specific functions

gpt-3.5-turbo-0613和gpt-4-0613的api是可以把函数描述放到请求中，然后让GPT帮忙生成执行函数需要的参数。注意这里只是让GPT帮忙生成函数需要的参数，需要自已调用该函数,GPT不会帮你调用。

### 六、Test changes systematically

当改变prompt时候，如何评估新的prompt是变好了还是变化了？有可能对于几个例子来说，该prompt是变好了，但是对于整体来讲是变差了。因此，有必须做一个全面的测试用例去评估当修改prompt后的性能。

好的评估准则

+ Representative of real-world usage (or at least diverse)
+ Contain many test cases for greater statistical power (see table below for guidelines)
|DIFFERENCE TO DETECT|AMPLE SIZE NEEDED FOR 95% CONFIDENCE|
|---|---|
|30%|~10|
|10%|~100|
|3%|~1,000|
|1%|~10,000|
+ Easy to automate or repeat

#### Evaluate model outputs with reference to gold-standard answers

参考标准答案评估模型输出。假设知道问题答案的**要点**，可以通过GPT去统计多少要点被命中。

下面是例子

```
# system
You will be provided with text delimited by triple quotes that is supposed to be the answer to a question. Check if the following pieces of information are directly contained in the answer: 

- Neil Armstrong was the first person to walk on the moon.
- The date Neil Armstrong first walked on the moon was July 21, 1969.

For each of these points perform the following steps:

1 - Restate the point.
2 - Provide a citation from the answer which is closest to this point.
3 - Consider if someone reading the citation who doesn't know the topic could directly infer the point. Explain why or why not before making up your mind.
4 - Write "yes" if the answer to 3 was yes, otherwise write "no".

Finally, provide a count of how many "yes" answers there are. Provide this count as {“count”: insert-count-here}.

# user

Neil Armstrong is famous for being the first human to set foot on the Moon. This historic event took place on July 21, 1969, during the Apollo 11 mission.

# gpt

1 - Neil Armstrong was the first person to walk on the moon.
2 - "Neil Armstrong is famous for being the first human to set foot on the Moon."
3 - Yes, someone reading this sentence could directly infer that Neil Armstrong was the first person to walk on the moon because it explicitly states that he was the first human to set foot on the Moon.
4 - Yes

1 - The date Neil Armstrong first walked on the moon was July 21, 1969.
2 - "This historic event took place on July 21, 1969, during the Apollo 11 mission."
3 - Yes, someone reading this sentence could directly infer that the date Neil Armstrong first walked on the moon was July 21, 1969 because it explicitly states that the historic event took place on that date.
4 - Yes

{"count": 2}
```

以上的例子中，两个要点均满足。下面是只满足一个要点的例子（**官方教程是基于GPT-4的，可能输出是只满足一个要点。我这里是基于GPT-3.5的，所以输出还是两个点都满足。由此可见，GPT-4的理解能力是高于GPT-3.5的，当然GPT-4的价格是GPT-3.5的20倍**）

```
# system
You will be provided with text delimited by triple quotes that is supposed to be the answer to a question. Check if the following pieces of information are directly contained in the answer: 

- Neil Armstrong was the first person to walk on the moon.
- The date Neil Armstrong first walked on the moon was July 21, 1969.

For each of these points perform the following steps:

1 - Restate the point.
2 - Provide a citation from the answer which is closest to this point.
3 - Consider if someone reading the citation who doesn't know the topic could directly infer the point. Explain why or why not before making up your mind.
4 - Write "yes" if the answer to 3 was yes, otherwise write "no".

Finally, provide a count of how many "yes" answers there are. Provide this count as {“count”: insert-count-here}.

# user

In the summer of '69, a voyage grand,
Apollo 11, bold as legend's hand.
Armstrong took a step, history unfurled,
"One small step," he said, for a new world.

# gpt （第二点GPT判断错误）

1 - Neil Armstrong was the first person to walk on the moon.
2 - "Armstrong took a step, history unfurled"
3 - Yes, someone reading this line could directly infer that Armstrong was the first person to walk on the moon because it mentions Armstrong taking a step, which implies that he was the first to do so.
4 - yes

1 - The date Neil Armstrong first walked on the moon was July 21, 1969.
2 - "In the summer of '69"
3 - Yes, someone reading this line could directly infer that the date of Armstrong's moonwalk was in the summer of 1969, which includes July, because it specifically mentions the summer of '69.
4 - yes

{"count": 2}
```

下面的例子是比较候选答案和参考答案的重叠，并且查看候选答案和参考答案是否冲突。

```
# system
Use the following steps to respond to user inputs. Fully restate each step before proceeding. i.e. "Step 1: Reason...".
    
Step 1: Reason step-by-step about whether the information in the submitted answer compared to the expert answer is either: disjoint, equal, a subset, a superset, or overlapping (i.e. some intersection but not subset/superset).
    
Step 2: Reason step-by-step about whether the submitted answer contradicts any aspect of the expert answer.
    
Step 3: Output a JSON object structured like: {"type_of_overlap": "disjoint" or "equal" or "subset" or "superset" or "overlapping", "contradiction": true or false}.

# user

Question: """What event is Neil Armstrong most famous for and on what date did it occur? Assume UTC time."""
    
Submitted Answer: """Didn't he walk on the moon or something?"""
    
Expert Answer: """Neil Armstrong is most famous for being the first person to walk on the moon. This historic event occurred on July 21, 1969."""

# gpt

Step 1: The information in the submitted answer is overlapping with the expert answer. Both mention Neil Armstrong walking on the moon.

Step 2: The submitted answer does not contradict any aspect of the expert answer.

Step 3: {"type_of_overlap": "overlapping", "contradiction": false}

```

下面的是有冲突例子，但是**同样因为chatgpt-3.5的关系，没有识别到该冲突**。Expert Answer是第一在月球上走的人，Submitted Answer是第二个在月球上走的人。
```
# system
Use the following steps to respond to user inputs. Fully restate each step before proceeding. i.e. "Step 1: Reason...".
    
Step 1: Reason step-by-step about whether the information in the submitted answer compared to the expert answer is either: disjoint, equal, a subset, a superset, or overlapping (i.e. some intersection but not subset/superset).
    
Step 2: Reason step-by-step about whether the submitted answer contradicts any aspect of the expert answer.
    
Step 3: Output a JSON object structured like: {"type_of_overlap": "disjoint" or "equal" or "subset" or "superset" or "overlapping", "contradiction": true or false}.

# user

Question: """What event is Neil Armstrong most famous for and on what date did it occur? Assume UTC time."""

Submitted Answer: """On the 21st of July 1969, Neil Armstrong became the second person to walk on the moon, following after Buzz Aldrin."""

Expert Answer: """Neil Armstrong is most famous for being the first person to walk on the moon. This historic event occurred on July 21, 1969."""

# gpt

Step 1: The information in the submitted answer compared to the expert answer is overlapping. Both answers mention Neil Armstrong walking on the moon on July 21, 1969.

Step 2: The submitted answer does not contradict any aspect of the expert answer.

Step 3: {"type_of_overlap": "overlapping", "contradiction": false}

```

## 参考资料

1. [promptingguide](www.promptingguide.ai)
2. [openai official tutorial](https://platform.openai.com/docs/guides/gpt-best-practices)
3. [吴恩达关于大模型的系列课程](https://learn.deeplearning.ai/)
4. [本地生活场景下的探索之如何写好Prompt？](https://mp.weixin.qq.com/s/vNycIeXGwAFmIU4BHsXLlQ)
5. [面向开发者的 LLM 入门课程](https://github.com/datawhalechina/prompt-engineering-for-developers)
