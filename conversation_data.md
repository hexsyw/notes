# 对话语料管理

最近想找个平台管理模型训练数据的语料。一是构造的 prompt 模板，我希望能够同时管理 instructions (或者说 system_prompt)、tools、还有对话模板；二是像管理代码一样管理这些数据，同时最好有可视化的能力。

实践了一些方案后，目前选择了 [LangSmith](https://www.langchain.com/langsmith)，它是 LangChain 生态里的一部分，结合 LangChain 试下来感觉还挺顺手的。


## Conversation

首先是对话语料，和 instructions 无关的数据。我习惯把对话语料分成 ``query`` 与 ``history`` 两部分，做成一个 jsonl，然后使用的时候转成 ``messages``。之前总是用自己写的包，现在用 LangChain 也可以快速实现。

这里用了 openai 的格式做了示例。

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder


template = ChatPromptTemplate.from_messages(
    [
        ("system", "You are a helpful assistant."),
        MessagesPlaceholder("history"),
        ("user", "{{query}}"),
    ],
    template_format="mustache",
)

history = [
    {"role": "user", "content": "hello_1"},
    {"role": "assistant", "content": "hello_2"},
]
query = "hello_3"

prompt = template.invoke({"history": history, "query": query})
print(prompt.to_string())
"""Output:

System: You are a helpful assistant.
Human: hello_1
AI: hello_2
Human: hello_3
"""

# 整个对话转成输入的 messages 也比较方便
import json
from langchain_core.messages.utils import convert_to_openai_messages

oai_messages = convert_to_openai_messages(prompt.to_messages())
print(json.dumps(oai_messages, ensure_ascii=False))

"""Output:

[{"role": "system", "content": "You are a helpful assistant."}, {"role": "user", "content": "hello_1"}, {"role": "assistant", "content": "hello_2"}, {"role": "user", "content": "hello_3"}]
"""
```


## Prompt Management

然后是把对话语料变成模型生成（与训练）需要的 prompt。比如，我个人比较习惯把 ``instructions`` 放在 ``system`` 中，这时候使用 ``langchain_core`` 的 prompt template 中的 ``system`` 换掉就行。同理，如果习惯把 ``instructions`` 放在 ``user`` 或者 ``huamn`` 里面，使用模板也比较方便。

然后，可以使用 langsmith 提交 prompt，进行管理。

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langsmith import Client


template = ChatPromptTemplate.from_messages(
    [
        ("system", "You are a helpful assistant."),
        MessagesPlaceholder("history"),
        ("user", "{{query}}"),
    ],
    template_format="mustache",
)

client = Client()

prompt_id = "pe_demo"
client.push_prompt(prompt_id, object=template)
```

如果你有 tools，则需要绑定一个模型。

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_openai import ChatOpenAI
from langsmith import Client


template = ChatPromptTemplate.from_messages(
    [
        ("system", "You are a helpful assistant."),
        MessagesPlaceholder("history"),
        ("user", "{{query}}"),
    ],
    template_format="mustache",
)

model = ChatOpenAI(model="gpt-4o-mini")

tools = [
    {
        "type": "function",
        "name": "get_horoscope",
        "description": "Get today's horoscope for an astrological sign.",
        "parameters": {
            "type": "object",
            "properties": {
                "sign": {
                    "type": "string",
                    "description": "An astrological sign like Taurus or Aquarius",
                },
            },
            "required": ["sign"],
        },
    },
]

client = Client()

chain = template | model.bind_tools(tools)

prompt_id = "pe_demo"
client.push_prompt(prompt_id, object=chain)
```

感觉如果能够单独管理自定义的 tools 就更好了。
