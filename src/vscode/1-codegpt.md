原文标题：CodeGPT: The VSCode Extension with ChatGPT-Like Functionalities

原文地址：<https://medium.com/geekculture/codegpt-the-vscode-extension-with-chatgpt-like-functionalities-783323a916c3>

原文作者：[The PyCoach](https://medium.com/@frank-andrade?source=post_page-----783323a916c3--------------------------------)

---

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d24e9dca8204686967029b84b6120a4~tplv-k3u1fbpfcp-watermark.image?)
<p align=center>Image made on Canva</p>

虽然我不是VSCode的粉丝，但是我必须承认VSCode比其他编辑器拥有更多更好用的扩展，其中之一就是CodeGPT。

CodeGPT允许我们通过官方OpenAI API在VSCode中使用GPT-3，使得我们可以在代码编辑器拥有了ChatGPT。

借助 CodeGPT，我们可以生成代码、解释代码、重构代码等等。

下面让我们看看如何设置CodeGPT并探索它的所有功能。

# VSCode中设置CodeGPT

打开VSCode编辑器，点击扩展，然后在搜索框中输入“Code GPT”，然后点击“安装”。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ccb52939117a47bfa4e28812d6c73e06~tplv-k3u1fbpfcp-watermark.image?)

Code GPT安装完，进到“Settings”界面，然后找到扩展下的“Code GPT”（macOS系统下，点击菜单栏的Code菜单，然后选择"Preferences"子菜单，选择“Settings”选项）。

CodeGPT的设置界面如下：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0f497ceef30b4323b90bc7ca03476d59~tplv-k3u1fbpfcp-watermark.image?)
想要让CodeGPT运行起来，需要在上图的第一个输入框处输入[OpenAI API key](https://beta.openai.com/account/api-keys)，API key需要到OpenAI的官网生成。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bd2c00df7b96490c8d8f4ad900e30de4~tplv-k3u1fbpfcp-watermark.image?)
其他输入项都可以采用默认设置，如果想要自定义设置，采用如下方式：

-   Max tokens: 您希望为 API 的每个请求获取的最大令牌数
-   Model: 目前此扩展中有 3 个模型可用 (text-davinci-003, code-cushman-001 and code-davinci-002)。在这些选项中code-davinci-002 在编码方面是最强大的。
-   Temperature: 控制输出中的随机性 ([temperature越低，GPT-3 就越有可能选择出现概率更高的词](https://algowriting.medium.com/gpt-3-temperature-setting-101-41200ff0d0be))

# 探索CodeGPT

以下是 CodeGPT 的主要功能：

-   Generate Code（生成代码）：您只需要输入一条注释，要求输出特定代码，然后按cmd-shift-i 组合键，然后 CodeGPT 将打开一个包含代码的新窗口。

![0_qr7OlOdRIQAEUV5S.gif](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5e0f41698cc44adeb60bcc0dac65923b~tplv-k3u1fbpfcp-watermark.image?)
<p align=center>来源: CodeGPT</p>

-   Explain Code（解释代码）: 选择某段代码，然后右键单击并选择“Explain CodeGPT”选项。 CodeGPT 将打开一个新窗口并解释代码。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc23cd2f79a24cdbab9d731018da76d2~tplv-k3u1fbpfcp-watermark.image?)
<p align=center>来源: CodeGPT</p>

结果如下：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fdfcb1ebe4754fb0b464fb67a60c4477~tplv-k3u1fbpfcp-watermark.image?)
<p align=center>来源: CodeGPT</p>

但上述功能还不是全部！ CodeGPT还有如下功能：

-   Ask questions（提问）
-   Refactor code（重构代码）
-   Document code（文档代码）
-   Find problems in your code（查找代码的Bug）

关于Code GPT的更多信息，点击[这里](https://marketplace.visualstudio.com/items?itemName=DanielSanMedium.dscodegpt&ssr=false#overview)
