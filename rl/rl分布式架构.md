
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

看来看去，基本都是大同小异，总结一下：

（1）是否需要ps：不需要的时候，参数在多个learner之间可能需要通过allreduce平均。

（2）并行：
- actor可以仅仅做跟环境交互采样，然后把样本直接发给learner（on-policy） 或者 放到replay memory（off-policy）然后由learner训练。
- actor可以做训练，也就是跟learner放到一起，训练得到的梯度就可以发给ps来更新。
- actor与learner之间，可以是一对多，可以是多对多。learner之间可以有通信，actor通常是相互独立的。

（3）cpu or gpu：
 - on-policy 一般使用cpu训练，因为每次训练的batch小。off-policy与之相反。
 - on-policy也可以使用gpu训练，那就需要每次将多个actor的样本攒起来训练一次（cpu采样 -> gpu前向+gpu反向，cpu采样+cpu前向 -> gpu反向）。

（4）同步 or 异步：如果on-policy采用异步，就相当于样本变成了off-policy，可以通过importance sampling作修正。

（5）单机 or 多机：单机可以省去通信。还有就是看扩展性，比如多个learner、或者采用ps。

（6）异步带来的效果问题：
 - 稳定性：参数加版本，过滤异常梯度。
 - importance sampling：解决actor使用旧策略产出样本的问题，对其作修正。

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
 - on-policy：没有replay memory。（也就是说引入很多的并行的agent同时与环境交互，可以达到跟replay memory一样的效果）
 - 与gorila不同，是单机多线程训练（hogwild）。
 - 每个worker包含一个actor、learner、local buffer，每个worker都独立地进行环境交互与模型训练。每个worker将梯度传给一个global network，然后这个global network会取平均更新模型。更新完模型再把这个模型参数传给所有的worker。

算法流程如下：

![image](https://user-images.githubusercontent.com/12492564/149653084-0ab84e8c-9c47-4c10-b8a8-3fd3a2d3118c.png)


### G-A3C

<img width="1072" alt="image" src="https://user-images.githubusercontent.com/12492564/153919189-56222d0b-845a-41af-9ed2-9170bdb23d23.png">

CPU只负责环境交互、GPU负责网络推理/学习，这是一个非常合理的资源分配，为大规模强化学习提供了可能性。模型网络只有一套，存储在GPU上。

不同环境的Agent产生的交互数据先放到一个Training Queue里，由Trainer先batch一下然后送到GPU去训练。与此同时，每个环境中的Agent要采取什么策略的输出请求也要先排在一个Prediction Queue里等待着由Predictor batch一下然后送到GPU去预测。可以看出当前预测采用的参数可能还是没来及更新的，有一定的延迟，不是严格的on-policy。

### DPPO

PPO（Proximal Policy Optimization）：是 policy gradient 的一个变形，通过重要性采样的方法，将on-policy转为off-policy。

policy gradient 是一个会花很多时间来采样数据的算法，大多数时间都在采样数据，agent 去跟环境做互动以后，接下来就要更新参数。你只能更新参数一次。接下来你就要重新再去收集数据， 然后才能再次更新参数。这显然是非常花时间的，所以我们想要从 on-policy 变成 off-policy。 这样做就可以用另外一个 policy， 另外一个 actor θ′ 去跟环境做互动(θ′被固定了)。用θ′收集到的数据去训练θ。

KL 散度： actor的动作上的差距，而不是它们参数上的差距。

![image](https://user-images.githubusercontent.com/12492564/154100327-e0ec9e37-1578-441d-abb4-88615d212404.png)

PPO的实现：它先初始化一个 policy 的参数θ0。然后在每一个迭代里面，你要用参数 θk， θk就是你在前一个训练的迭代得到的 actor 的参数，你用它去跟环境做互动，采样到一大堆状态-动作的对。
然后你根据互动的结果，估测一下 A。然后你就使用 PPO 的优化的公式。但跟原来的 policy gradient 不一样，原来的 policy gradient 只能更新一次参数，更新完以后，你就要重新采样数据。但是现在不用，你拿  θk去跟环境做互动，采样到这组数据以后，你可以让 θ 更新很多次，想办法去最大化目标函数。这边 θ 更新很多次没有关系，因为我们已经有做重要性采样，所以这些经验，这些状态-动作的对是从θk
采样出来的没有关系。θ 可以更新很多次，它跟θk变得不太一样也没有关系，你还是可以照样训练θ。

![image](https://user-images.githubusercontent.com/12492564/154102497-18f3dd18-3854-4346-b0b7-3722a8cdfc4c.png)

PPO 算法有两个主要的变种：PPO-Penalty 和 PPO-Clip。

PPO-Penalty

![image](https://user-images.githubusercontent.com/12492564/154103571-14c42ab4-f2b2-4d54-87e5-2f7bcad46349.png)

在这个方法里面，你先设一个你可以接受的 KL 散度的最大值。假设优化完这个式子以后，你发现 KL 散度的项太大，那就代表说后面这个惩罚的项没有发挥作用，那就把 β 调大。
另外，你设一个 KL 散度的最小值。如果优化完上面这个式子以后，你发现 KL 散度比最小值还要小，那代表后面这一项的效果太强了，你怕他只弄后面这一项，那 θ 跟 θk 
都一样，这不是你要的，所以你要减少β。

PPO-Clip

![image](https://user-images.githubusercontent.com/12492564/154103979-21d70016-4980-4247-943a-e4484687c433.png)

clip如下，也就是既不让它太大或者太小

![image](https://user-images.githubusercontent.com/12492564/154104407-66b65630-f003-4ad7-8994-9056a3999783.png)


keras实现ppo-clip：[https://keras.io/examples/rl/ppo_cartpole/](https://keras.io/examples/rl/ppo_cartpole/)
