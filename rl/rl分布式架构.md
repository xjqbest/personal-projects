
5个基本概念：
 - env
 - state
 - reward
 - agent
 - policy


工程实现的几个组成部分：
 - env：环境
 - actor：负责agent与环境交互，得到reward和新的state
 - learner：负责更新policy的参数
 - policy：决定了agent要采取什么action

几种并行方式：
 - actor并行：多个actor跟环境交互
 - learner并行：多个learner训练更新参数

架构演进：DQN -> GORILA -> A3C -> Ape-x -> IMPALA

### DQN

![image](https://user-images.githubusercontent.com/12492564/149626533-35d78385-fa08-40b7-93e3-95d6dd8df0c1.png)

支持actor并行。

引入ps后，可以支持learner并行，也就是多个learner并行训练push梯度然后同步参数。这种方式下，actor和learner在同一个节点。

![image](https://user-images.githubusercontent.com/12492564/149627427-a3ba0f77-e59e-4090-a43c-362290d8f7fa.png)

### GORILA

![image](https://user-images.githubusercontent.com/12492564/149627797-84abc9c3-3f17-4d9b-b81d-530ae94392b2.png)


特点：
 - 多个actor跟环境交互产生样本放到replay memory（actor与learner之间提供和接收样本，有local和global两种方式）。
 - 使用ps保存更新参数，多个learner并行训练push梯度然后同步参数。
 - actor和learner中的参数与ps同步：Q网络每次action前同步，target Q网络定期同步。
 - bundled mode：也就是最简单的方式，是每个actor-learner-replay memory作为一组绑定起来。
 - 稳定性：参数保存version避免参数版本过于旧；检查梯度，如果其loss过大则舍弃。

算法流程如下：

![image](https://user-images.githubusercontent.com/12492564/149628373-1ca1fb92-3eeb-406f-8d9b-3eea32812c00.png)

一个例子如下：

![image](https://user-images.githubusercontent.com/12492564/149628441-4e333865-50d2-41a8-a268-b630501bfe4c.png)

# A3C

![image](https://user-images.githubusercontent.com/12492564/149652662-3dfb8941-8161-464c-92e6-ad69e10eb01b.png)

A3C采用actor-critic算法框架，特点是：
 - on-policy：没有replay memory。
 - 与gorila不同，没单机多线程训练（hogwild）。
 - 每个worker包含一个actor、learner、local buffer，每个worker都独立地进行环境交互与模型训练。每个worker将梯度传给一个global network，然后这个global network会取平均更新模型。更新完模型再把这个模型参数传给所有的worker。

算法流程如下：

![image](https://user-images.githubusercontent.com/12492564/149653084-0ab84e8c-9c47-4c10-b8a8-3fd3a2d3118c.png)


