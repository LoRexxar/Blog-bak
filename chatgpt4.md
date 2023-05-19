---
title: 从0到1的ChatGPT - 进阶篇（四）- 训练自己的ChatGPT
date: 2023-05-19 16:09:22
tags:
- chatgpt
- llm
---

在之前的文章中曾经提到过，ChatGPT其实是不接受来自互联网的知识的，他的所有内容都是来自于至少3年前各种来源的知识库。**但这并不意味着ChatGPT没有能力学习你的回答**。

首先ChatGPT一般会根据你和他的问答内容进行一定的上下文参考，其次，由于ChatGPT学习的内容之庞大，你通过一种直白的方式问不到的答案不一定是他不会，有可能是你问的方式不对。

在ChatGPT的官方文档中，**他首先鼓励你通过提供多个示例来让ChatGPT更准确的寻找答案，他把这个方案称之为****"few-shot learning."**

除此之外，当然他也允许你通过**微调功能来对ChatGPT进行一定的训练**，来获得一个更符合自己要求的ChatGPT，当然，这个功能是收费的。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202305191612484.png)

但Fine-tuning这个功能目前只能应用于**GPT3的基础模型**，就目前而言，**这个功能其实还不如很多市面上的其他大模型，openai并没有给出特别好的自定义方案给大家。但这篇文章还是先聊聊这个。**

# 通过微调ChatGPT训练

## 准备工作

首先你需要在openai的api基础上操作，所以你需要一个简单的openai环境。

```plain
pip install --upgrade openai
```

当然你需要提前配置openai api key，这个key可以在openai的平台后台获得，这里就不多说了。

```plain
export OPENAI_API_KEY="<OPENAI_API_KEY>"
```

## 准备训练数据

首先我们需要准备相应的训练数据，这个数据文件都必须是JSONL文件，每行都是一个提示对，类似于

```plain
{"prompt": "<prompt text>", "completion": "<ideal generated text>"}
{"prompt": "<prompt text>", "completion": "<ideal generated text>"}
{"prompt": "<prompt text>", "completion": "<ideal generated text>"}
...
```

一般来说，你提供的训练示例最好有几百个，训练数据会直接影响到最终模型的质量。

你可以用openai提供的工具来验证和处理。

```plain
openai tools fine_tunes.prepare_data -f <LOCAL_FILE>
```

你可以提供**CSV, TSV, XLSX, JSON**,**JSONL**格式的训练数据

## 创建微调模型

在准备好相应的训练数据之后，你可以用opanai的工具来创建微调后的模型。

```plain
openai api fine_tunes.create -t <TRAIN_FILE_ID_OR_PATH> -m <BASE_MODEL>
```

当然，这里指定的基础模型只包含GPT3的部分，包括**ada**, **babbage**, **curie**,**davinci**

当然由于这个功能并不是在本地完成的，在openai的平台中可能会排在几小时之后。你可以随时中断这个任务。并随时恢复进程。

```plain
openai api fine_tunes.follow -i <YOUR_FINE_TUNE_JOB_ID>
```

在成功训练完成之后，你会获得相应的模型id。你就可以通过对应的模型id来使用它。

当然你也可以随时删除这些模型。

```plain
openai api models.delete -i <FINE_TUNED_MODEL>
```

## 一些训练范例

我研究了一些相应的训练范例实践，其中还有很多有意思的方案。我挑了一些比较有特点的选出来。

**1、否定训练**

如果你在和ChatGPT的对话当中，遇到反馈的事实错误，你可以**通过否定训练来排除这部分并更正**

```plain
{"prompt":"testtest", "completion":" yes"}
{"prompt":"test", "completion":" no"}
```

**2、情感分析**

在ChatGPT的配置中，有个很重要的参数就是情绪值。很显然，ChatGPT的情绪肯定不是空穴来风，这本身是基于数据集训练的结果。

当然，你也可以通过**微调来对你数据集标注情绪以此训练**

```plain
{"prompt":"Overjoyed with the new iPhone! ->", "completion":" positive"}
{"prompt":"@lakers disappoint for a third straight night  ->", "completion":" negative"}
```

你可以通过api来获取prompt对应的情绪判断值。

**3、分类**

如果你想要ChatGPT帮你完成**分类的工作**，那最好的方案是提供范例并以数字作为标志.

```plain
{"prompt":"test", "completion":" 1"}
{"prompt":"1231421", "completion":" 2"}
```

通过数字标志可以帮助ChatGPT更准确的对目标做分类。

**4、样本处理与提取**

如果你需要用ChatGPT来完成**样本提取**工作，你可以用一些简单的多行范例来举证。

```plain
{"prompt":
"Portugal will be removed from the UK's green travel list from Tuesday, amid rising coronavirus cases and concern over a \"Nepal mutation of the so-called Indian variant\". It will join the amber list, meaning holidaymakers should not visit and returnees must isolate for 10 days...\n\n###\n\n", 
"completion":
" Portugal\nUK\nNepal mutation\nIndian variant END"}
```

理论上来说，你可以提供大量的样本标准文本的提取方案。

**5、聊天机器人**

如果你需要完成一个聊天机器人的功能，最好的办法是给ChatGPT提供**问题以及大量回答样本**，这样可以让ChatGPT学习他应该回答的内容。

```plain
{"prompt":"Summary: <summary of the interaction so far>\n\nSpecific information:<for example order details in natural language>\n\n###\n\nCustomer: <message1>\nAgent: <response1>\nCustomer: <message2>\nAgent:", 
"completion":" <response2>\n"}
{"prompt":"Summary: <summary of the interaction so far>\n\nSpecific information:<for example order details in natural language>\n\n###\n\nCustomer: <message1>\nAgent: <response1>\nCustomer: <message2>\nAgent: <response2>\nCustomer: <message3>\nAgent:", 
"completion":" <response3>\n"}
```

你可以像这个范例中讲的一样，按照问题回答场景来划分提示词。

## 一个小小的实例

接下来跟着前面的每一步来训练一个自己的ChatGPT，首先我们需要准备一份数据集。**这里我选择用我的博客内容来做初步的内容训练。**

用一个简单的python3脚本来**处理所有的md文件并生成对应的jsonL文件**。

**这个prompt的范例比较粗暴，不是很靠谱的，只是测试一下。**

```python
import os
import glob
import re
import json
import codecs

folder_path = 'posts' # 指定文件夹路径
output_file = 'output.jsonl' # 指定输出文件名

md_files = glob.glob(os.path.join(folder_path, '*.md')) # 获取所有的md文件路径

with codecs.open(output_file, 'w', encoding='utf-8') as f:
    for file in md_files:
        with codecs.open(file, 'r', encoding='utf-8', errors='ignore') as md:
            text = md.read()
            match = re.search(r'title: (.+)\n', text) # 匹配标题和内容
            text = re.sub(r"```.*?```", "", text, flags=re.DOTALL)
            if match:
                i = 0
                max_length = 2000

                while len(text) > i*2000:
                    t = text[i*max_length:i*max_length+max_length]

                    prompt = match.group(1) + ' Part {}'.format(i+1)
                    completion = ' ' + t + 'END'
                

                    data = {"prompt": prompt, "completion": completion}
                    json_data = json.dumps(data) + '\n' # 将字典格式化为JSONL格式
                    f.write(json_data)

                    i += 1
```

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202305191612272.png)

然后我们用**openai来处理一下**这部分数据集

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202305191612678.png)

他会给你一些修改意见和处理方案，并且会自动处理一下你的数据集。

然后我们在**基础的4个GPT-3模型中选取一个作为基础模型，其中****davinci这个模型要相对来说更强大，也更适合进一步培养。**但要注意的是，davinci相比之下**贵10倍还多**。

```python
openai api fine_tunes.create -t .\output_prepared.jsonl -m davinci
```

要注意**这一步是要翻墙的**，不然无法上传文件。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202305191612115.png)

等待微调的任务处理完成。如果不小心中断，可以用follow继续

```python
openai api fine_tunes.follow -i ft-PcXP6lbEZKDHo3ez8986RWmZ
```

之后就是等待结果即可，我自己研究了一下发现这个东西有点儿贵的我训练集数据也就400多条，还用了比较便宜的curie模型，结果还花了10刀。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202305191612439.png)

训练完成之后你就可以使用这个模型来交互。但我研究了一下，**这个微调后的Chatgpt只能用Complete功能，你可以使用api或者platform来调用这个模型。**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202305191612388.png)

但还是那句话，这个方案问题相当之大，**一个是GPT3在现在的大模型中是比较菜的**，先不说GPT4，连3.5什么时候上线这个功能还遥遥无期，另一方面就现在的内容而言，**训练的结果和价格其实不太成正比**，一方面**微调这个功能很依赖训练的数据有效度**，你简单的拿一大堆数据来搞不但很贵还效果不好，你精心准备各种提示词和内容又**违背了本身依靠ai来做总结归纳的初心**，所以现在市面上更多的基于chatgpt的第三方工具，都是用了一些其他的方案。

# 基于上下文训练

在前面的文章中，其实有讲到这个关键点，虽然**ChatGPT不会学习来自互联网上的任何对话信息**，但为了保证对话的流畅性，**ChatGPT会记录每个对话session的上下文**并在这个基础上对你进行反馈。

所以就衍生了一种相关的方案，**通过储存上下文来实现简单的训练**，这个东西最大的优势是**可以使用现在大模型中最领先的GPT4模型**，而问题是，这种方案只能**实现特别简单的训练**，尤其是**不能太多条+长度过长**！

很多第三方的ChatGPT和一些浏览器插件其实都实现了类似的功能。这里拿我使用的第三方ChatGPT来看看这个功能。

- https://github.com/Yidadaa/ChatGPT-Next-Web

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202305191612021.png)

首先是你可以在对话之前对其预设面具，这个东西可以有很多预设内容。这里我们只假设了**一个比较简单的基础设定**。当然这只是一个简单的人设面具，你也可以通过特别具体的prompt来作为基础了，这部分内容在前面的文章中讲到过。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202305191612272.png)

我们再回到魔兽的小助手上，我们再提供一下相关的数据。**把相关的数据以及条件放在方案预设之中**。这里提前准备好相应的数据内容。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202305191612818.png)

通过**设置前置上下文，可以在一定程度上影响ChatGPT的功能以及表现**，来实现一个简单的自定义ChatGPT。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202305191612385.png)

除此之外，[ChatGPT-Next-Web](https://github.com/Yidadaa/ChatGPT-Next-Web)本身也有**消息摘要功能**，会总结前几条发送的内容摘要。附加到请求中。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202305191612304.png)

要注意的是，**这个方案更适用于和ChatGPT本身功能类似的场景，大部分是文字类相关的，主要是设定场景和人设。在一定的限定场景下并控制回复，就比如非常经典的文案辅助、批改作文等等。**

但单纯的预制prompt不适用于**有大量基础数据的特殊模型**，当然，魔高一尺道高一丈。也有不少产品用了一些旁敲的方案。还是拿刚才基于博客文章训练的问答机器人来举例子。

如果单纯的靠文章数据总结或者干脆直接拿博客文章来训练，这份**数据集很大而且内容冗杂**，直接训练的效果很不好，所以更靠谱的方案是，在**ChatGPT前面挂一个数据库。**

用户输入问题的时候可以**简单的拆解关键词然后从数据库查询结果，然后再作为上下文传到ChatGPT，并由ChatGPT做总结和摘要。**

但这种基于上下文的训练方案问题比较多，专业性越强的效果就会越差，内容越多效果也会越差，所以**这其实也算是一种临时方案，与实际训练过的效果差很多。**

# 基于其他LLM大模型的训练方案

其实抛开ChatGPT以外，现在市面上还有**非常多比较靠谱的LLM大模型**，虽然和GPT4都有很大的差距，但能比拟GPT3.5的大模型已经相当多了。

我思来想去，感觉这个问题还比较多，这部分内容我打算专门拆到另一篇文章里再说。
