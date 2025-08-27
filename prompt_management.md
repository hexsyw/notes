# Prompt Management

今天在一篇 blog 的安利下，转投了 ``langfuse`` 的怀抱。

我对于 **prompt management** （或者说语料管理）的主要需求来自构造训练数据。训练任务和训练数据多了以后，难免会出现“这个数据我之前做过，但是我忘记放在哪里了”的情况。

特别是随着模型基础能力的逐渐变强，post-training 阶段的数据也逐渐变得精细、复杂。两三年前那种一个 prompt 大力出一批的方法已经不太适用了。

我日常使用的数据结构其实就四个部分：**query**, **history**, **instructions (system)**, **tools**。有时候还会有其他的 **context**，但总体来说不多。

我理想中的 prompt management 工具就是一个 ``name`` 或者 ``id`` 能够获取对应的 prompt template，然后把 **query**, **history**, **instructions**（还有 **tools** 和 **context**）这些变量很方便的替换进去，继而我得到了我想要的 ``input``。这样我不管事构造数据还是管理训练数据，整个 datapoint 的结构都会变得清爽很多。

之前试了 ``langsmith``，整体感觉虽然还行，但仍然会觉得不够简洁干练。相比之下，``langfuse`` 的整个设计更符合我个人的审美一些。

首先是对话语料。

```python
from langfuse import get_client
 
langfuse = get_client()
 
prompt = langfuse.create_prompt(
    name="query-history-demo",
    type="chat",
    prompt=[
      { "role": "system", "content": "You are a helpfule assistant."},
      { "type": "placeholder", "name": "history" },
      { "role": "user", "content": "{{query}}" },
    ],
    labels=["production"],  # directly promote to production
)

messages = prompt.compile(
    query="Hello world",
    history=[
        {"role": "user", "content": "I love Ron Fricke movies like Baraka"},
        {"role": "user", "content": "Also, the Korean movie Memories of a Murderer"}
    ]
)
```

基本和 ``langsmith`` 的设计差不多，但是省去了一些选项（如我其实只想固定使用 mustache 的 ``template_format``）。
转成我常用的 messages 格式，也不需要额外引入函数。

另外，``langfuse`` 支持在 prompt 中存储一个自由度极高的 ``config``，可以把 **model**, **tools**, **context** 等等都用这个 ``config`` 管理。比 ``langsmith`` 需要绑定一个“模型”更符合我的需求。

官方文档给的例子：

```python
langfuse.create_prompt(
    name="story_summarization",
    prompt="Extract the key information from this text and return it in JSON format. Use the following schema: {{json_schema}}",
    config={
        "model":"gpt-3.5-turbo-1106",
        "temperature": 0,
        "json_schema":{
            "main_character": "string (name of protagonist)",
            "key_content": "string (1 sentence)",
            "keywords": "array of strings",
            "genre": "string (genre of story)",
            "critic_review_comment": "string (write similar to a new york times critic)",
            "critic_score": "number (between 0 bad and 10 exceptional)"
        }
    },
    labels=["production"]
);
```

最后，对于两个工具对不支持单独管理 ``tools``，我反思了一下，其实 ``tool`` 这种东西作为程序代码实现的某种抽象，和 prompt 这种几乎纯文本的东西还在规范格式上有很大的区别的。使用者完全可以把 ``tools`` 当做类似代码进行管理，而 prompt 更多时候更像一种数据。
