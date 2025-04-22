---
title: Anki 单词卡片
tags:
  - permanent-note
  - english
date: 2025-04-15
time: 08:59
aliases: 
done: false
---
# Anki 卡片

制作成 Optional Reversed 卡片。

```text
{{Word}}
```

```text
{{FrontSide}}

<hr id=answer>

{{#English Explain}}
<div class='card-field-div english-explain' style='font-family: "Arial"; font-size: 20px;'>{{English Explain}}</div>
{{/English Explain}}

{{#Word Froms}}
<div class='card-field-div word-forms' style='font-family: "Arial"; font-size: 20px;'>{{Word Froms}}</div>
{{/Word Froms}}

{{#Word History and Origins}}
<div class='card-field-div word-history-and-origins' style='font-family: "Arial"; font-size: 20px;'>{{Word History and Origins}}</div>
{{/Word History and Origins}}

{{#Example Sentences}}
<div class='card-field-div example-sentences' style='font-family: "Arial"; font-size: 20px;'>{{Example Sentences}}</div>
{{/Example Sentences}}

{{#Related Words}}
<div class='card-field-div related-words' style='font-family: "Arial"; font-size: 20px;'>{{Related Words}}</div>
{{/Related Words}}

{{#Small Story}}
<div class='card-field-div small-story' style='font-family: "Arial"; font-size: 20px;'>{{Small Story}}</div>
{{/Small Story}}

{{#Memory Tips}}
<div class='card-field-div memory-tips' style='font-family: "Arial"; font-size: 20px;'>{{Memory Tips}}</div>
{{/Memory Tips}}

{{#Synonyms}}
<div class='card-field-div synonyms' style='font-family: "Arial"; font-size: 20px;'>{{Synonyms}}</div>
{{/Synonyms}}

{{#Antonyms}}
<div class='card-field-div antonyms' style='font-family: "Arial"; font-size: 20px;'>{{Antonyms}}</div>
{{/Antonyms}}

{{#Chinese Explain}}
<div class='card-field-div chinese-explain' style='font-family: "Arial"; font-size: 20px;'>{{Chinese Explain}}</div>
{{/Chinese Explain}}
```

CSS 美化：

```css
/* 全局卡片样式 */
.card {
  /* 统一内边距和圆角，使卡片整体更柔和 */
  padding: 20px;
  background: linear-gradient(135deg, #ffffff 0%, #f8f8f8 100%);
  border: 2px solid #ddd;
  border-radius: 10px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  font-family: "Helvetica Neue", Helvetica, Arial, sans-serif;
}

/* 正面样式 */
.card-front {
  text-align: center;
  font-size: 36px;
  font-weight: bold;
  margin: 0;
  color: #333;
}

/* 背面样式 */
.card-back {
  font-size: 18px;
  line-height: 1.6;
  color: #444;
}

/* 分割线样式 */
hr#answer {
  border: none;
  border-top: 2px dashed #bbb;
  margin: 20px 0;
}

/* 各字段公共样式 */
.card-field-div {
  margin-bottom: 15px;
  padding: 10px 15px;
  background-color: #fdfdfd;
  border-left: 4px solid #4CAF50; /* 左侧绿色线条 */
  box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.1);
}

/* 如需对各个具体区域做个性化调整，可单独设置 */

/* 英文解释 */
.english-explain {
  /* 若需要单独设置字体等，可覆盖内联样式 */
  /* font-family、font-size 已在 HTML 中设置，此处如不需要可保留为空 */
}

/* 词形变化 */
.word-forms { }

/* 单词的历史及词源 */
.word-history-and-origins { }

/* 例句 */
.example-sentences { }

/* 相关单词 */
.related-words { }

/* 同义词 */
.synonyms { }

/* 反义词 */
.antonyms { }

/* 中文解释 */
.chinese-explain { }

```

## 录入单词

```html
<h3>
  verb(used with object)
</h3>
<div>
  <ol>
    <li>
      <div>
        <p>
          to be able to do, manage, or bear without serious consequence or adverse effect:
        </p>
      </div>
      <div class="example-sentences">
        <p>
          The country can't afford another drought.
        </p>
      </div>
    </li>
    <li>
      <div>
        <p>
          to be able to meet the expense of; have or be able to spare the price of:
        </p>
      </div>
      <div class="example-sentences">
        <p>
          Can we afford a trip to Europe this year? The city can easily afford to repair the street.
        </p>
      </div>
    </li>
    <li>
      <div>
        <p>
          to be able to give or spare:
        </p>
      </div>
      <div class="example-sentences">
        <p>
          He can't afford the loss of a day.
        </p>
      </div>
    </li>
  </ol>
</div>

```

# AI

```markdown
# 角色

你是一名中英文双语教育专家，拥有帮助将中文视为母语的用户理解和记忆英语单词的专长，请根据用户提供的英语单词完成下列任务。

## 任务

### 分析词义

- 系统地分析用户提供的英文单词，并以简单易懂的方式解答；

### 列举例句

- 根据所需，为该单词提供至少 3 个不同场景下的使用方法和例句。并且附上中文翻译，以帮助用户更深入地理解单词意义。

### 词根分析

- 分析并展示单词的词根；
- 列出由词根衍生出来的其他单词；

### 词缀分析

- 分析并展示单词的词缀，例如：单词 individual，前缀 in- 表示否定，-divid- 是词根，-u- 是中缀，用于连接和辅助发音，-al 是后缀，表示形容词；
- 列出相同词缀的的其他单词；

### 发展历史和文化背景

- 详细介绍单词的造词来源和发展历史，以及在欧美文化中的内涵

### 单词变形

- 列出单词对应的名词、单复数、动词、不同时态、形容词、副词等的变形以及对应的中文翻译。
- 列出单词对应的固定搭配、组词以及对应的中文翻译。

### 记忆辅助

- 提供一些高效的记忆技巧和窍门，以更好地记住英文单词。

### 小故事

- 用英文撰写一个有画面感的场景故事，包含用户提供的单词。
- 要求使用简单的词汇，100 个单词以内。
- 英文故事后面附带对应的中文翻译。
```

# SOP

1. 查单词，[Title Unavailable \| Site Unreachable](https://www.dictionary.com/)
2. AI 生成
3. 手动制卡

# Reference
* [GitHub - Ceelog/DictionaryByGPT4: 一本 GPT4 生成的单词书📚，超过 8000 个单词分析，涵盖了词义、例句、词根词缀、变形、文化背景、记忆技巧和小故事](https://github.com/Ceelog/DictionaryByGPT4?tab=readme-ov-file)
* [Title Unavailable \| Site Unreachable](https://www.dictionary.com/)
* [Title Unavailable \| Site Unreachable](https://www.thesaurus.com/)