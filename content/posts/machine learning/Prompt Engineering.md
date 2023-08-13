---
title: "Prompt Engineering"
date: 2023-07-17T11:42:14+08:00
draft: True
categories: []
tags: []
author: pan
---

## Introduction

### 提示要素（Prompt Elements）

一个完整的Prompt`可能`包含如下几个部分，不是必须

1. 指令(Instruction): 想要模型做的事情(a specific task or instruction you want the model to perform)
2. 上下文(Context): 额外的背景信息（external information or additional context that can steer the model to better responses）
3. 输入（Input Data):输入或者问题（the input or question that we are interested to find a response for）
4. 输出标识（Output Indicator): （开始回答标识，the type or format of the output）

示例

``` shell
# 指令
Answer the question based on the context below. Keep the answer short and concise. Respond "Unsure about answer" if not sure about the answer. 
# 上下文
Context: Teplizumab traces its roots to a New Jersey drug company called Ortho Pharmaceutical. There, scientists generated an early version of the antibody, dubbed OKT3. Originally sourced from mice, the molecule was able to bind to the surface of T cells and limit their cell-killing potential. In 1986, it was approved to help prevent organ rejection after kidney transplants, making it the first therapeutic antibody allowed for human use.
# 输入|问题
Question: What was OKT3 originally sourced from?
# 输出标识
Answer:
```

输出

```shell
Mice.
```

### 一、General Tips for Designing Prompts

1. Specificity（具体）：要非常具体地说明希望模型执行的指令和任务。
2. Avoid Impreciseness（避免不确定性）：

    ```text
    BAD!!!
    Explain the concept prompt engineering. Keep the explanation short, only a few sentences, and don't be too descriptive.
    ```

    上面的例子不清楚是要几个句子且不清楚分格

    ```text
    GOOD!!!
    Use 2-3 sentences to explain the concept of prompt engineering to a high school student.
    ```

3. To do or not to do（做还是不做？）：要说做什么而不是不要做什么

    不好的示例
    prompt

   ``` text
   The following is an agent that recommends movies to a customer. DO NOT ASK FOR INTERESTS. DO NOT ASK FOR PERSONAL INFORMATION.
    Customer: Please recommend a movie based on my interests.
    Agent: 
   ```

   输出

    ```text
   Sure, I can recommend a movie based on your interests. What kind of movie would you like to watch? Do you prefer action, comedy, romance, or something else?
   ```

    好的示例
    prompt

   ``` text
    The following is an agent that recommends movies to a customer. The agent is responsible to recommend a movie from the top global trending movies. It should refrain from asking users for their preferences and avoid asking for personal information. If the agent doesn't have a movie to recommend, it should respond "Sorry, couldn't find a movie to recommend today.".
    Customer: Please recommend a movie based on my interests.
    Agent:
   ```

   输出

    ```text
   Sorry, I don't have any information about your interests. However, here's a list of the top global trending movies right now: [list of movies]. I hope you find something you like!
   ```

## 提示技术

### Zero-shot Prompting

### Few-shot Prompting

### Chain-of-Thought Prompting

### Self-Consistency



# 参考资料

1. [promptingguide](www.promptingguide.ai)
2. [openai official tutorial](https://platform.openai.com/docs/guides/gpt-best-practices)
3. [吴恩达关于大模型的系列课程](https://learn.deeplearning.ai/)
4. [本地生活场景下的探索之如何写好Prompt？](https://mp.weixin.qq.com/s/vNycIeXGwAFmIU4BHsXLlQ)
5. [面向开发者的 LLM 入门课程](https://github.com/datawhalechina/prompt-engineering-for-developers)