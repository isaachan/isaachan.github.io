---
title: 利用大模型构建免费的“看图片讲故事”应用程序
description: 本文介绍了一个“看图片讲故事”应用程序的构建过程。该程序结合transformers和langchain库，以及huggingface的免费模型（以及为了更好的结果而使用了收费的chatGPT），能根据图片生成并朗读小故事。这个案例展示了人工智能在创意写作和语音合成方面的应用，为开发者提供了将复杂技术转化为互动应用的案例。
categories:
 - software
tags:
---

陪孩子玩儿的时候，我经常用手边的玩具或小物件编故事讲给他听，这激发了我将langchain、大型语言模型（LLM）和huggingface技术融入一个小应用的想法。这个应用能够根据图片生成故事并朗读出来，为孩子带来新奇的故事体验。

本文讲述了构建这个“看图片讲故事”应用程序的过程。该程序结合transformers和langchain库，以及huggingface的免费模型，能根据图片生成并朗读小故事。当然，在文本生成的任务上，我也比较了免费的和收费的LLM的效果，最终还是选择了chatGPT。如果只是处于练习和学习的目的，免费的模型也是完全胜任的。最终，这个案例展示了人工智能在创意写作和语音合成方面的应用，为开发者提供了将复杂技术转化为互动应用的案例。

整个过程分为三步：

- 将图片转换为一句话描述
- 根据一句话描述编写童话故事
- 将童话故事文本转换为语音文件进行播放

这三步分别用到了不同的模型。开始吧。

# 将图片转换为一句话描述

首先，我希望这个程序可以根据我随手拍的小玩具或者小物件，可以生成一句话的描述。这里涉及到“Image-to—Text”的模型的能力。于是，我们来到[HuggingFace](https://huggingface.co/)社区，找到[tasks的分类页面](https://huggingface.co/tasks)，在“Multimodel”下找到“Image-to-Text”并进入，里面有300多个用从图片生成文字的模型。有些免费的模型，是可以直接在右边栏上测试效果的。
![image](/images/2023-12-28-build-story-teller/hf-image-to-text-model.png)
*Hugging Face上的Image-to-Text模型*

我试过了几个模型，发现目前排名的第一的来自于Salesforce的blip-image-captioning-base效果还可以。下面我们就尝试使用这个模型来实现总结图片内容的功能。由于模型是在HuggingFace上，我们需要用的transformers库。先来安装它，(关于transformers可以阅读[这篇文章](https://huggingface.co/docs/transformers/v4.18.0/en/quicktour))

```bash
pip3 install transformers
```

HuggingFace上blip-image-captioning-base的页面提供了生成图片描述的实例代码，可以直接拿过用，

```python
from transformers import pipeline

image_url = "xxxx"
captioner = pipeline("image-to-text",model="Salesforce/blip-image-captioning-base")
captioner(image_url)
caption = captioner[0]['generated_test']
print(f"The caption of the given picture: [{caption}]")
```
下面我用下面的图片作为输入，来测试上面的代码的执行效果，

![image](/images/2023-12-28-build-story-teller/car.jpg)

得到了如下的结果，

```bash
python3 story-generator.py ~/car.jpg
The caption of the given picture: [a model car on a table]
```

"a model car on a table" - 这个结果挺准确的。

# 根据一句话描述编写童话故事

接下来就把“a model car on a table”这句话扩展为一则童话故事。这种“命题作文”是大语言模型典型的原生能力，因此完成这个并不难，有很多LLM可供选择。我先后尝试了三个模型，分别是Llama2，chatGLM3和chatGPT，最后还是选择了chatGPT。

## Llama2 vs. chatGLM3 vs. chatGPT

简单对比一下这三个LLM在处理这个小任务上的表现吧。首先，由于Llama2和chatGLM3都是开源的大模型，所以必须部署在本地，动辄几个GB的显存和显卡的需求，是目前个人使用它们最大的障碍。这也是我最后选择了chatGPT的主要原因。

下面具体在讲一下Llama2和chatGLM3在我本地环境的表现。我是用一台Macbook Air (16GB内存，M1) 来运行的Llama2-7b，模型执行的速度还可以，基本上在1分钟内可以生成结果，但是有两个问题，

1. Llama2训练用的中文语料不多，所以生成的中文并不是很自然（尤其和chatGPT的结果对比后更加明显）
2. 由于第一步得到的描述“a model car on a table”是英文，而7b模型的能力有限，无法把生成的英文内容翻译为中文

![image](/images/2023-12-28-build-story-teller/llama2-7b-output.png)
*Llama2-7b生成的故事文本*

而chatGLM3模型是中国的智谱和清华大学联合开发的，对于中文的处理要好很多。但是chatGLM的问题在于本地运行消耗的资源太高。为了运行chatGLM3，我使用了自己另一台搭载32GB内存的linux laptop，没有显存和显卡，所以用CPU来运行，大概要花3-4分钟到时间才能得到文本。虽然文本的质量有所提高，但是可用性太差了。

最后，我只得放弃了在本地运行开源大模型的想法，转而使用chatGPT了。

## 使用chatGPT+LangChain生成故事文本

虽然openai的APIAPI不算难用，但是我还是选择了当下流行的LangChain，除了LangChain更好用之外，也为以后可以轻松切换到其他模型做准备。

实现功能的代码并不复杂，只用了LangChain的PromoptTemplate和LLMChain两个组件，

```python
def generate_story(scenario):
    template = """
    你是一位很会讲故事的人，下面的Context是一句英文，请你根据这句话讲一个幽默诙谐的故事。要求用中文，并且字数控制在100字以内。
    
    CONTEXT: {scenario}
    STORY:
    """

    prompt = PromptTemplate(template=template, input_variables=["scenario"])
    story_llm = LLMChain(llm=ChatOpenAI(model_name="gpt-3.5-turbo", temperature=1), 
                         prompt=prompt,
                         verbose=True)

    story = story_llm.predict(scenario=scenario)
    return story
```

这段代码用了PromptTemplate给LLM提供提示词前缀，模型选择了"gpt-3.5-turbo"，为了让故事更有趣，把temperature设为了最高值1。

用第一步的输出为参数，调用generate_story函数，


```bash
python story-generator.py ./car.jpg 

a model car on a table
Here is the story:

> Entering new LLMChain chain...
Prompt after formatting:
    你是一位很会讲故事的人，下面的Context是一句英文，请你根据这句话讲一个幽默诙谐的故事。要求用中文，并且字数控制在100字以内。

    CONTEXT: a model car on a table
    STORY:

> Finished chain.
曾经有一辆模型车在桌子上，它自诩拥有无与伦比的驾驶技巧...
```

chatGPT生成的这个故事如下，还挺有趣的，而且结尾竟然还上价值了:)

> 曾经有一辆模型车在桌子上，它自诩拥有无与伦比的驾驶技巧。一天，它向附近的水杯挑衅：“我才不害怕你！”接者，它大摇大摆地驶向杯子。可就在距离杯子仅剩一毫米的时候，模型车突然紧张起来，因为它忘记了没有驾驶员！咚！模型车敲击在杯子上，糗态百出。从此以后，每次骄傲的模型车依然自诩非凡但怎样驾驶都无济于事。因为无论多么强大的马力，没有一个人执掌方向，车辆也只能在原地打转了。

# 将童话故事文本转换为语音文件进行播放

将文本转化为语音的任务称为TTS (Text-to-Speech)，这是比大模型要早也成熟的多的技术。不过，出于学习的目的，我依然尝试使用huggingface上的免费的模型来完成这件事。

虽然TTS很成熟，但是Huggingface上免费的午餐并不是很多，

![image](/images/2023-12-28-build-story-teller/tts-models.png)

我按照顺序，依次试了几个，要么收费、要么不支持中文、要么shutting down了。直到第六个模型才真的可用。

这个叫做“espnet/kan-bayashi_ljspeech_vits”的模型没有提供本地部署的方法，只能通过API来调用，好在并不复杂。代码如下：

```python
def text_to_speech(story):
    API_TOKEN = "[HG_API_TOKEN]"
    API_URL = "https://api-inference.huggingface.co/models/espnet/kan-bayashi_ljspeech_vits"
    headers = {"Authorization": f"Bearer {API_TOKEN}"}
    payload = { "inputs": story }
    repsonse = requests.post(API_URL, headers=headers, json=payload)
    if repsonse.status_code == 200:
        with open("audio.flac", "wb") as f:
            f.write(repsonse.content)
        print("Audio saved to audio.flac")
    else:
        print(f"Something went wrong: {repsonse.text}")
```

不过这个模型的性能并不算太好，100字的文本大概花了3-5分钟左右。除此之外，它还有两个问题，

1. 中文支持很差！听上去语素是用英文和粤语拼出来的（后面有链接，可以感受一下）
2. 只能生成flac格式的文件，如果要在大多数播放器上播放，需要转成mp3格式

下面就是espnet/kan-bayashi_ljspeech_vits模型生成故事音频文件，可以感受一下（一定要对着文本听，否则真的很难听懂）：

<audio controls>
  <source src="/audios/a-car-on-a-table.mp3" type="audio/mpeg">
  Your browser does not support the audio element.
</audio>

# 完整代码

```python
import dotenv
dotenv.load_dotenv()

from transformers import pipeline
from langchain.chains import LLMChain
from langchain.prompts import PromptTemplate
from langchain_community.chat_models import ChatOpenAI
import requests
import sys

def img_to_text(url):
    p = pipeline("image-to-text", model="Salesforce/blip-image-captioning-base", max_new_tokens=40)
    text = p(url)[0]["generated_text"]
    return text

def generate_story(scenario):
    template = """
    你是一位很会讲故事的人，下面的Context是一句英文，请你根据这句话讲一个幽默诙谐的故事。要求用中文，并且字数控制在100字以内。
    
    CONTEXT: {scenario}
    STORY:
    """

    prompt = PromptTemplate(template=template, input_variables=["scenario"])
    story_llm = LLMChain(llm=ChatOpenAI(model_name="gpt-3.5-turbo", temperature=1), 
                         prompt=prompt,
                         verbose=True)

    story = story_llm.predict(scenario=scenario)
    return story

def text_to_speech(story):
    API_TOKEN = "[HG_API_TOKEN]"
    API_URL = "https://api-inference.huggingface.co/models/espnet/kan-bayashi_ljspeech_vits"
    headers = {"Authorization": f"Bearer {API_TOKEN}"}
    payload = { "inputs": story }
    repsonse = requests.post(API_URL, headers=headers, json=payload)
    if repsonse.status_code == 200:
        with open("audio.flac", "wb") as f:
            f.write(repsonse.content)
        print("Audio saved to audio.flac")
    else:
        print(f"Something went wrong: {repsonse.text}")

if len(sys.argv) > 1:
    url = sys.argv[1]
else:
    url = input("Please input the image url: ")

text = img_to_text(url)
print(text)

print("Here is the story: ")
story = generate_story(text)
print(story)

text_to_speech(story)
```

# 总结
在这个案例中，我使用了transformers和langchain，用了三四个免费/收费的模型，花了两小时左右，实现了一个简单但有用的小应用程序，展示了人工智能在创意写作和语音合成方面的新的开发模式。可以预想在不久的将来，拥有一些编程能力的产品经理可以以非常低的成本构建AI应用的原型了。

未来已来，只是尚未流行。

