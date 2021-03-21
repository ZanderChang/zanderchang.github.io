---
title: Gym.spaces总结
mathjax: true
date: 2020-06-03 11:32:19
categories:
tags:
- gym
---

[源码](https://github.com/openai/gym/tree/master/gym/spaces)


<!-- more -->

||Box|Dict|Discrete|MultiBinary|MultiDiscrete|Tuple|
|-|-|-|-|-|-|-|
||连续空间|Space字典|离散空间$\{ 0, 1, \dots, n-1 \}$|多维01空间|多维离散空间（游戏控制器）|Space元组|
|示例|`Box(low=-1.0, high=2.0, shape=(3, 4), dtype=np.float32)`||`Discrete(2)`|`MultiBinary(5)`|`MultiDiscrete([ 5, 2, 2 ])`|`Tuple((spaces.Discrete(2), spaces.Discrete(3)))`|
|sample||||[0,1,0,1,0]||(0, 2)|

```py
# spaces.Dict
self.nested_observation_space = spaces.Dict({
    'sensors':  spaces.Dict({
        'position': spaces.Box(low=-100, high=100, shape=(3,)),
        'velocity': spaces.Box(low=-1, high=1, shape=(3,)),
        'front_cam': spaces.Tuple((
            spaces.Box(low=0, high=1, shape=(10, 10, 3)),
            spaces.Box(low=0, high=1, shape=(10, 10, 3))
        )),
        'rear_cam': spaces.Box(low=0, high=1, shape=(10, 10, 3)),
    }),
    'ext_controller': spaces.MultiDiscrete((5, 2, 2)),
    'inner_state':spaces.Dict({
        'charge': spaces.Discrete(100),
        'system_checks': spaces.MultiBinary(10),
        'job_status': spaces.Dict({
            'task': spaces.Discrete(5),
            'progress': spaces.Box(low=0, high=100, shape=()),
        })
    })
})
```

||Action|State|
|-|-|-|
|MountainCarContinuous-v0|`Box(1,)`|`Box(2,)`|
|[LunarLanderContinuous-v2](https://github.com/openai/gym/blob/3bd5ef71c2ca3766a26c3dacf87a33a9390ce1e6/gym/envs/box2d/lunar_lander.py)|`Box(2,)`|`Box(8,)`|