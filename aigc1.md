---
title: 打造自己的AIGC应用（一）入门篇
date: 2023-07-19 18:18:59
tags:
- aigc
---

其实细数AI的发展历程非常之久，而让AI的应用一下子出现在人们眼前的其实就是**ChatGPT的出现**，这意味着AIGC应用已经从概念为王变的非常实用了。伴随着ChatGPT的出现，大量的开源大模型也如雨后春笋一样出现。就现在而言，**打造一个自己的AIGC应用已经非常简单了**。

<!--more-->

# 基础环境

我们需要配置一个环境

- **python3.8+，不要太新**
- **CUDA+环境**
- **pytorch**
- **支持C++17的编译器**

首先我比较推荐你配置一个**anaconda**的环境，因为pytorch的其他安装方法真的很麻烦

- https://www.anaconda.com/distribution/#download-section

然后你需要安装CUDA的环境，正常来说只需要下载**对应的CUDA版本**即可

- https://developer.nvidia.com/cuda-downloads

然后就是安装**pytorch**的环境，这个环境比较麻烦，正常来说**通过conda来安装**是比较靠谱的办法，当然有些时候就是安不了。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822387.png)

如果安装不成功，就只能用源码来编译了。

- https://github.com/pytorch/pytorch#from-source

首先，clone一下源码

```python
git clone --recursive https://github.com/pytorch/pytorch
cd pytorch
# if you are updating an existing checkout
git submodule sync
git submodule update --init --recursive
```

然后安装一下对应的各种依赖

```python
conda install cmake ninja
# Run this command from the PyTorch directory after cloning the source code using the “Get the PyTorch Source“ section below
pip install -r requirements.txt

conda install mkl mkl-include
# Add these packages if torch.distributed is needed.
# Distributed package support on Windows is a prototype feature and is subject to changes.
conda install -c conda-forge libuv=1.39
```

然后**windows的源码编译有点儿复杂**，具体要参考各种情况下的编译

- https://github.com/pytorch/pytorch#install-pytorch

在编译这个**pytorch**这个东西的时候我遇到过贼多问题，其中大部分问题我都搜不到解决方案，最终找到的最靠谱的方案是，**不能用太低或者太高版本的python**，会好解决很多，最终我选择了用3.10版本的python，解决了大部分的问题。

另外就是如果gpu不是很好或者显存不是很高，也可以**使用cpu版本**，大部分电脑的内存都会比较大，起码能跑起来。

**如果是windows**，那一定会用到**huggingface**，有个东西我建议一定要注意下。

默认的huggingface和pytorch的缓存文件夹是在～/.cache/下，是在c盘下面，而一般LLM的模型文件都贼大，很容易把C盘塞满，这个注意要改下。

**在环境变量里加入****HF_HOME和TORCH_HOME ，设****置为指定变量即可。**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822422.png)

除此之外，有的项目也会提供**docker化的部署方案**，如果采用这种方案，就必须在宿主机安装**NVIDIA Container Toolkit**，并重启docker

```python
sudo apt-get install -y nvidia-container-toolkit-base
sudo systemctl daemon-reload 
sudo systemctl restart docker
```

# 进阶构成

## LLM

**LLM全称****Large Language Model**，大语言模型，是以ChatGPT为代表的ai对话核心模块，相比我们无法控制、训练的ChatGPT，也逐渐在出现大量的开源大语言模型，尤其是以**ChatGLM、LLaMA**为代表的轻量级语言模型相当好用。

虽然**这些开源语言模型相比ChatGPT差距巨大**，但深度垂直领域的ai应用也在逐渐被人们所认可。与其说我们想要在开源世界寻找ChatGPT的代替品，不如说这些开源大语言模型的出现，意味着我们有能力打造自己的GPT。

- **ChatGLM-6B**
- https://github.com/THUDM/ChatGLM-6B
- **ChatGLM2-6B**
- https://github.com/THUDM/ChatGLM2-6B

目前中文领域效果最好，也是应用最多的开源底座模型。大部分的**中文GPT二次开发几乎都是在这个模型的基础上做的开发**，尤其是2代之后进一步拓展了基座模型的上下文长度。最厉害的是它允许商用。

- **Moss**
- https://github.com/OpenLMLab/MOSS

**MOSS是一个支持中英双语和多种插件的开源对话语言模型**，moss-moon系列模型具有160亿参数，在FP16精度下可在单张A100/A800或两张3090显卡运行，在INT4/8精度下可在单张3090显卡运行。MOSS基座语言模型在约七千亿中英文以及代码单词上预训练得到，后续经过对话指令微调、插件增强学习和人类偏好训练具备多轮对话能力及使用多种插件的能力。

- **ChatRWKV**
- https://github.com/BlinkDL/ChatRWKV

**一系列基于RWKV架构的Chat模型（包括英文和中文）**，发布了包括Raven，Novel-ChnEng，Novel-Ch与Novel-ChnEng-ChnPro等模型，可以直接闲聊及进行诗歌，小说等创作，包括7B和14B等规模的模型。

LLM的基座模型说实话有点儿多，尤其是在最开始的几个开源之后，后面各种LLM基座就像雨后春笋一样出现了，**比较可惜的是目前的的开源模型距离ChatGPT的差距非常之大，大部分的模型只能达到GPT3的级别，距离GPT3.5都有几个量级的差距，更别提GPT4了。**

- https://github.com/HqWu-HITCS/Awesome-Chinese-LLM
- https://github.com/chenking2020/FindTheChatGPTer

给大家看一个通过**一些标准排名出来的LLM排行榜，这个说法比较多，一般就是看样本集的覆盖程度。**

- [**https://github.com/CLUEbenchmark/SuperCLUElyb**](https://github.com/CLUEbenchmark/SuperCLUElyb)

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822932.png)

## Embedding 

Embedding 模型也是GPT的很重要一环，在之前的文章里曾经提到过。由于GPT的只能依赖对话的模式受限于上下文的长度。

所以也就衍生出了不少的开源Embedding模型

- https://huggingface.co/GanymedeNil/text2vec-large-chinese
- https://huggingface.co/shibing624/text2vec-base-chinese

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822989.png)

## gradio

**gradio是一个非常有名的机器学习用于数据演示的web框架。**通过gradio可以快速的构建一个可以实时交互的web界面。有点儿像flask

- https://github.com/gradio-app/gradio

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822258.png)

首先要注意gradio最起码**python在3.8版本以上**.

```python
import gradio as gr

def greet(name):
    return "Hello " + name + "!"

demo = gr.Interface(fn=greet, inputs="text", outputs="text")
    
demo.launch()
```

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822400.png)

gradio支持非常多这类的常用场景，就比如**文本、勾选框、输入条，甚至文件上传、图片上传**，都有非常不错的原生支持。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822513.png)

## FastChat

**FastChat是一个在LLM基础上构筑的一体化平台**，FastChat是基于**LLaMA**做的二次调参训练。

```python
pip3 install fschat
```

正常使用需要用机器生成**Vicuna**模型，将LLaMa weights合并Vicuna weights。而这个过程需要**大量的内存和CPU**，官方给出的参考数据是

- Vicuna-7B：30 GB of CPU RAM
- Vicuna-13B：60 GB of CPU RAM

**如果没有足够的内存用**，可以尝试下面两个办法来操作一下：

1、在命令中加入**--low-cpu-mem**，这个命令可以把峰值内存降到16G以下

2、创建一个比较大的交换分区让操作系统用硬盘作为虚拟内存

```python
python3 -m fastchat.model.apply_delta \
    --base-model-path /path/to/llama-7b \
    --target-model-path /path/to/output/vicuna-7b \
    --delta-path lmsys/vicuna-7b-delta-v1.1

python3 -m fastchat.model.apply_delta \
    --base-model-path /path/to/llama-13b \
    --target-model-path /path/to/output/vicuna-13b \
    --delta-path lmsys/vicuna-13b-delta-v1.1
```

下载完模型文件之后，可以快捷的使用对应的模型

```python
python3 -m fastchat.serve.cli --model-path lmsys/fastchat-t5-3b-v1.0
```

相比其他的基座模型LLM，**FastChat的平台化**程度就比较高了。

首先提供了**controller和model worker**分别部署的方案，一对多的方案本身比较符合项目化的结构。

```python
python3 -m fastchat.serve.controller

python3 -m fastchat.serve.model_worker --model-path /path/to/model/weights
```

而且同样**利用gradio构建了相应的web界面**

```python
python3 -m fastchat.serve.gradio_web_server
```

除此之外FastChat还提供了**和openai完全兼容的api接口**和restfulapi.

```python
import openai
openai.api_key = "EMPTY" # Not support yet
openai.api_base = "http://localhost:8000/v1"

model = "vicuna-7b-v1.3"
prompt = "Once upon a time"

# create a completion
completion = openai.Completion.create(model=model, prompt=prompt, max_tokens=64)
# print the completion
print(prompt + completion.choices[0].text)

# create a chat completion
completion = openai.ChatCompletion.create(
  model=model,
  messages=[{"role": "user", "content": "Hello! What is your name?"}]
)
# print the completion
print(completion.choices[0].message.content)
```

甚至可以直接以api的方式接入到其他的平台中，完成度很高。

## 知识库文件

**知识库文件是langchain类方案中比较重要的一环**，所有的问题会先进入知识库中搜索结果然后再作为上下文，知识库文件的数据量会直接影响到这类应用的结果有效度。而现在**比较常见的相似度检测用的都是faiss**，构建向量数据库用于数据比对。

- https://github.com/facebookresearch/faiss

| 知识库数据                                                  | FAISS向量                                                    |
| ----------------------------------------------------------- | ------------------------------------------------------------ |
| 中文维基百科截止4月份数据，45万                             | 链接：https://pan.baidu.com/s/1VQeA_dq92fxKOtLL3u3Zpg?pwd=l3pn 提取码：l3pn |
| 截止去年九月的130w条中文维基百科处理结果和对应faiss向量文件 | 链接：https://pan.baidu.com/s/1Yls_Qtg15W1gneNuFP9O_w?pwd=exij 提取码：exij |
| 💹 [大规模金融研报知识图谱](http://openkg.cn/dataset/fr2kg)  | 链接：https://pan.baidu.com/s/1FcIH5Fi3EfpS346DnDu51Q?pwd=ujjv 提取码：ujjv |

相应的现在很多应用还内置了**用于测试知识库的接口**，比如langchain-ChatGLM

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822165.png)

通过**微调知识相关度的阈值**，可以让回答消息更有效。你甚至可以直接在平台新建知识库并录入数据。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822043.png)

# langchain

**langchain是现在成熟度比较高的一套Aigc应用，现在比较主流的一种知识库检索方案**，用的是曾经的文章中提到过的基于上下文的训练方案，用户输出会先**进入数据库检索**，然后找出最匹配问题的部分结果然后和问题**一起加入到prompt的上下文**中，最终由**LLM生成最终的回答**。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822632.png)

这个方案是目前**最经典的知识库型训练方案**，最有效的解决了**大模型本身训练的难度和反馈结果的有效度**难以兼容的问题。

- https://github.com/hwchase17/langchain

建立在langchain的思想上，其实衍生了非常多比较有意思的项目，**一方面引用了包括ChatGLM-6B等各种开源的大模型**，也用了**开源的embedding方案**来处理文本。

- https://github.com/THUDM/ChatGLM-6B

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822033.png)

- https://huggingface.co/GanymedeNil/text2vec-large-chinese/tree/main

另一方面呢也做了比较成熟的**vue前端+知识库**，可以快速的拼凑出可用的chat ai。

- https://github.com/imClumsyPanda/langchain-ChatGLM
- https://github.com/yanqiangmiffy/Chinese-LangChain#![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822670.png)

## langchain-ChatGLM

langchain-ChatGLM是诸多langchain方案中中文支持实现的比较好的一个，过程包括**加载文件 -> 读取文本 -> 文本分割 -> 文本向量化 -> 问句向量化 -> 在文本向量中匹配出与问句向量最相似的****top k****个 -> 匹配出的文本作为上下文和问题一起添加到****prompt****中 -> 提交给****LLM****生成回答**。

整个项目中的每个部分都可以一定程度的自由组合，Embedding 默认选用的是 [**GanymedeNil/text2vec-large-chinese**](https://huggingface.co/GanymedeNil/text2vec-large-chinese/tree/main)，LLM 默认选用的是 [**ChatGLM-6B**](https://github.com/THUDM/ChatGLM-6B)。或者也可以通过**fastchat**来接入。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822380.png)

配置完成并安装好环境之后，就可以运行，首次运行会下载对应的大模型。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822386.png)

当然，这个模型实在是太大了，命令行下载的时候非常容易出问题，所以可以参考ChatGLM-6B的方案，自己下载模型然后再加载。

- [https://github.com/THUDM/ChatGLM-6B#%E4%BB%8E%E6%9C%AC%E5%9C%B0%E5%8A%A0%E8%BD%BD%E6%A8%A1%E5%9E%8B](https://github.com/THUDM/ChatGLM-6B#从本地加载模型)

比较靠谱的就是先下模型实现，然后再单独下载模型并覆盖所有的文件

```python
GIT_LFS_SKIP_SMUDGE=1 git clone https://huggingface.co/THUDM/chatglm-6b
```

- https://cloud.tsinghua.edu.cn/d/fb9f16d6dc8f482596c2/

然后需要修改对应的配置文件中模型的位置，在configs/model_config.py

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822879.png)

默认的知识库文件路径是

```python
knowledge_base\samples
```

如果想用自用的本地知识文件，放在对应目录的knowledge_base即可。现在的知识库文件会遍历目录下的文件，所以指定目录即可。

```python
python cli_demo.py
```

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822803.png)

```python
python3.10 .\webui.py
```

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202307191822855.png)
