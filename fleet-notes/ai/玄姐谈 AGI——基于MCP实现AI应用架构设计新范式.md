---
title: 玄姐谈 AGI——基于MCP实现AI应用架构设计新范式
tags:
  - fleet-note
  - llm
  - ai/mcp
date: 2025-04-27
time: 20:03
aliases: 
is_archie: false
---


![image.png](https://images.hnzhrh.com/note/20250427200722306.png)
24 年单智能体
25 年多智能体
25 年+通用多智能体


![image.png](https://images.hnzhrh.com/note/20250427201407322.png)


![image.png](https://images.hnzhrh.com/note/20250427202411784.png)

MCP 就是个代理


![image.png](https://images.hnzhrh.com/note/20250427202739632.png)
![image.png](https://images.hnzhrh.com/note/20250427203115310.png)
MCP Tool 是代理的最小单位，比如代理数据读，代理某个接口的访问等等。

![image.png](https://images.hnzhrh.com/note/20250427203325820.png)

在第一步中，需要把 MCP 整个的一些描述，包括有哪些 MCP Server、Tools + 用户的提示词都需要喂给 LLM，利用大模型的推理能力，判断需要调用那个 MCP 的 Server，需要调用哪个 Tool

![image.png](https://images.hnzhrh.com/note/20250427204418462.png)

![image.png](https://images.hnzhrh.com/note/20250427204538762.png)

如果有 100 w 个 MCP Server 该怎么办？ LLM 的 context 限制了。

MCP 是否需要 Fuction Calling？是

![image.png](https://images.hnzhrh.com/note/20250427205328464.png)

如果 LLM 没有 FC，MCP 喂给 LLM 是没有这个能力的，不会吐出来该调用哪个 MCP Server，如果没有 Fuction Calling，需要微调。Fuction Calling 是大模型预训练的能力。


![image.png](https://images.hnzhrh.com/note/20250427205854928.png)

怎么落地？

![image.png](https://images.hnzhrh.com/note/20250427210511746.png)

AI 网关解决大模型能力不足，大模型选择

MCP 网关

传统流量网关

![image.png](https://images.hnzhrh.com/note/20250427211039644.png)


AI 网关能力：
![image.png](https://images.hnzhrh.com/note/20250427211329637.png)
MCP 网关

![image.png](https://images.hnzhrh.com/note/20250427211514310.png)

基于 MCP 的 AI 应用架构设计新范式

![image.png](https://images.hnzhrh.com/note/20250427211729104.png)
![image.png](https://images.hnzhrh.com/note/20250427211804749.png)
![image.png](https://images.hnzhrh.com/note/20250427212117081.png)


![image.png](https://images.hnzhrh.com/note/20250427212242863.png)


![image.png](https://images.hnzhrh.com/note/20250427212523844.png)



# Reference