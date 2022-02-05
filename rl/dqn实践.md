```python
import gym
import random
import numpy as np
import math
import copy
from functools import reduce
from tensorflow.keras import models, layers, optimizers

global_config = {
    "env" : "CartPole-v1",
    "batch_size" : 64,
    "replay_capacity" : 10000,
    "max_episode": 500,
    "max_step": 500,
    "mlp_hidden_dim": 128,
    "mlp_lr": 0.001,
    "target_update_freq": 4,
    "gamma": 0.95,
}


class ENV:
    def __init__(self):
        self._env = gym.make(global_config["env"])
        self._env.reset()
        print("env action_space", self._env.action_space)
        print("env observation_space", self._env.observation_space)
        print("env reward_range", self._env.reward_range)
        self._action_dim = self._env.action_space.n
        self._state_dim = reduce(lambda x, y: x * y, self._env.observation_space.shape)

    def reset(self):
        return self._env.reset()

    # action: 0 left 1 right
    def step(self, action):
        return self._env.step(action)

    def render(self):
        self._env.render()

    def close(self):
        self._env.close()


class ReplayBuffer:
    def __init__(self):
        self._buffer = []
        self._capacity = global_config["replay_capacity"]
        self._index = 0
        self._batch_size = global_config["batch_size"]

    def store(self, ins):
        if len(self._buffer) < self._capacity:
            self._buffer.append(ins)
        else:
            self._buffer[self._index % self._capacity] = ins
            self._index += 1

    def sample(self):
        if len(self._buffer) == 0:
            raise RuntimeError("ReplayBuffer is empty")
        batch = random.sample(self._buffer, self._batch_size)
        state, action, reward, next_state, done =  zip(*batch)
        return np.array(state), np.array(action), np.array(reward), np.array(next_state), np.array(done)

    def size(self):
        return len(self._buffer)

class DQN:
    def __init__(self):
        self._batch_size =  global_config["batch_size"]
        self._max_episode = global_config["max_episode"]
        self._max_step = global_config["max_step"]
        self._replay_buffer = ReplayBuffer()
        self._env = ENV()
        self._hidden_dim = global_config["mlp_hidden_dim"]
        self._lr = global_config["mlp_lr"]
        self._gamma=  global_config["gamma"]
        self._target_update_freq = global_config["target_update_freq"]
        self._epsilon = lambda step_idx: 0.01 + (0.9 - 0.01) * math.exp(-1.0 * step_idx / 500)
        self._target_net = self.net()
        self._policy_net = self.net()

    def net(self):
        model = models.Sequential()
        model.add(layers.Dense(self._hidden_dim, input_dim=self._env._state_dim, activation='relu'))
        model.add(layers.Dense(self._hidden_dim, input_dim=self._hidden_dim, activation='relu'))
        model.add(layers.Dense(self._env._action_dim, input_dim=self._hidden_dim, activation='relu'))
        model.compile(loss='mean_squared_error', optimizer=optimizers.Adam(self._lr))
        return model

    def act(self, state, step):
        if random.random() > self._epsilon(step):
            #print("state",state)
            action = np.argmax(self._policy_net.predict(np.array([state]))[0])
        else:
            action = random.randrange(self._env._action_dim)
        return action

    def update(self):
        if self._replay_buffer.size() < self._batch_size:
            return
        state_batch, action_batch, reward_batch, next_state_batch, done_batch = self._replay_buffer.sample()
        q_batch = self._policy_net.predict(state_batch)
        q_next_batch = self._target_net.predict(next_state_batch)

        for i in range(self._batch_size):
            a = action_batch[i]
            q_batch[i][a] = reward_batch[i] + self._gamma * np.amax(q_next_batch[i][a]) * (1 - done_batch[i])

        self._policy_net.fit(state_batch, q_batch)

    def train(self):
        total_step = 0
        for episode in range(1, self._max_episode + 1):
            state = self._env.reset()
            episode_reward = 0
            for step in range(0, self._max_step):
                total_step += 1
                action = self.act(state, total_step)
                next_state, reward, done, _ = self._env.step(action)
                self._env.render()
                self._replay_buffer.store(copy.deepcopy([state, action, reward, next_state, done]))
                state = copy.deepcopy(next_state)
                self.update()
                episode_reward += reward
                if done:
                    break

            if episode % self._target_update_freq == 0:
                self._target_net.set_weights(self._policy_net.get_weights())
            print("episode", episode, "reward", episode_reward)
        self._policy_net.save("my_model")
        print("finish")
        self._env.close()

if __name__ == "__main__":

    dqn = DQN()
    dqn.train()
```
