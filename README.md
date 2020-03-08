# An Optimistic Perspective on Offline Reinforcement Learning

This project provides the open source implementation using the
[Dopamine][dopamine] framework for running experiments mentioned in [An Optimistic Perspective on Offline Reinforcement Learning][paper].
In this work, we use the logged experiences of a DQN agent for training off-policy
agents (shown below) in an offline setting (*i.e.*, [batch RL][batch_rl]) without any new
interaction with the environment during training. Refer to
[offline_rl.github.io][project_page] for the project page.

<img src="https://i.imgur.com/Ntgcecq.png" width="95%"
     alt="Architechture of different off-policy agents" >

[paper]: https://arxiv.org/pdf/1907.04543.pdf
[dopamine]: https://github.com/google/dopamine


## Atari-Replay Dataset (Logged DQN data) 

The atari-replay-dataset dataset was collected as follows:
We first train a [DQN][nature_dqn] agent, on all 60 [Atari 2600 games][ale]
with [sticky actions][stochastic_ale] enabled for 200 million frames (standard protocol) and save all of the experience tuples
of *(observation, action, reward, next observation)* (approximately 50 million)
encountered during training. 

This logged DQN data can be found in the public [GCP bucket][gcp_bucket]
`gs://atari-replay-datasets` which can be downloaded using [`gsutil`][gsutil].
To install gsutil, follow the instructions [here][gsutil_install].

After installing gsutil, run the command to copy the entire dataset:

```
gsutil -m cp -R gs://atari-replay-datasets/dqn
```

To run the dataset only for a specific Atari 2600 game (*e.g.*, replace `GAME_NAME`
by `Pong` to download the logged DQN replay datasets for the game of Pong),
run the command:

```
gsutil -m cp -R gs://atari-replay-datasets/dqn/[GAME_NAME]
```


This data can be generated by running the online agents using
[`batch_rl/baselines/train.py`](https://github.com/google-research/batch_rl/blob/master/batch_rl/baselines/train.py) for 200 million frames
(standard protocol). Note that the dataset consists of approximately 50 million
experience tuples due to frame skipping (*i.e.*, repeating a selected action for
`k` consecutive frames) of 4. The stickiness parameter is set to 0.25, *i.e.*,
there is 25% chance at every time step that the environment will execute the
agent's previous action again, instead of the agent's new action.

[nature_dqn]: https://www.nature.com/articles/nature14236?wm=book_wap_0005
[gsutil_install]: https://cloud.google.com/storage/docs/gsutil_install#install
[gsutil]: https://cloud.google.com/storage/docs/gsutil
[batch_rl]: http://tgabel.de/cms/fileadmin/user_upload/documents/Lange_Gabel_EtAl_RL-Book-12.pdf
[stochastic_ale]: https://arxiv.org/abs/1709.06009
[ale]: https://github.com/mgbellemare/Arcade-Learning-Environment
[gcp_bucket]: https://console.cloud.google.com/storage/browser/atari-replay-datasets
[project_page]: http://offline-rl.github.io

## Asymptotic Performance of offline agents on Atari-replay dataset

<div>
  <img src="https://i.imgur.com/gAWGgJx.png" width="49%" 
       alt="Number of games where a batch agent outperforms online DQN">
  <img src="https://i.imgur.com/QJiCg37.png" width="49%" 
       alt="Asymptotic Performance of offline agents on DQN data">
</div>

## Installation
Install the dependencies below, based on your operating system, and then
install Dopamine, *e.g*.

```
pip install git+https://github.com/google/dopamine.git
```

Finally, download the source code for batch RL, *e.g.*

```
git clone https://github.com/google-research/batch_rl.git
```

### Ubuntu

If you don't have access to a GPU, then replace `tensorflow-gpu` with
`tensorflow` in the line below (see [Tensorflow
instructions](https://www.tensorflow.org/install/install_linux) for details).

```
sudo apt-get update && sudo apt-get install cmake zlib1g-dev
pip install absl-py atari-py gin-config gym opencv-python tensorflow-gpu
```

### Mac OS X

```
brew install cmake zlib
pip install absl-py atari-py gin-config gym opencv-python tensorflow
```

## Running Tests

Assuming that you have cloned the
[batch_rl](https://github.com/google-research/batch_rl.git) repository,
follow the instructions below to run unit tests.

#### Basic test
You can test whether basic code is working by running the following:

```
cd batch_rl
python -um batch_rl.tests.atari_init_test
```

#### Test for training an agent with fixed replay buffer
To test an agent using a fixed replay buffer, first generate the data for the
Atari 2600 game of `Pong` to `$DATA_DIR`.

```
export DATA_DIR="Insert directory name here"
mkdir -p $DATA_DIR/Pong
gsutil -m cp -R gs://batch-rl-datasets/dqn/Pong/1 $DATA_DIR/Pong
```

Assuming the replay data is present in `$DATA_DIR/Pong/1/replay_logs`, run the `FixedReplayDQNAgent` on `Pong` using the logged DQN data:

```
cd batch_rl
python -um batch_rl.tests.fixed_replay_runner_test \
  --replay_dir=$DATA_DIR/Pong/1
```

## Training batch agents on DQN data

The entry point to the standard Atari 2600 experiment is
[`batch_rl/fixed_replay/train.py`](https://github.com/google-research/batch_rl/blob/master/batch_rl/fixed_replay/train.py).
Run the batch `DQN` agent using the following command:

```
python -um batch_rl.fixed_replay.train \
  --base_dir=/tmp/batch_rl \
  --replay_dir=$DATA_DIR/Pong/1 \
  --gin_files='batch_rl/fixed_replay/configs/dqn.gin'
```

By default, this will kick off an experiment lasting 200 training iterations
(equivalent to experiencing 200 million frames for an online agent).

To get finer-grained information about the process,
you can adjust the experiment parameters in
[`batch_rl/fixed_replay/configs/dqn.gin`](https://github.com/google-research/batch_rl/blob/master/batch_rl/fixed_replay/configs/dqn.gin),
in particular by increasing the `FixedReplayRunner.num_iterations` to see
the asymptotic performance of the batch agents. For example,
run the batch `REM` agent for 800 training iterations on the game of Pong 
using the following command:

```
python -um batch_rl.fixed_replay.train \
  --base_dir=/tmp/batch_rl \
  --replay_dir=$DATA_DIR/Pong/1 \
  --agent_name=multi_head_dqn \
  --gin_files='batch_rl/fixed_replay/configs/rem.gin' \
  --gin_bindings='FixedReplayRunner.num_iterations=1000' \
  --gin_bindings='atari_lib.create_atari_environment.game_name = "Pong"'
```

More generally, since this code is based on Dopamine, it can be
easily configured using the
[gin configuration framework](https://github.com/google/gin-config).


## Dependencies

The code was tested under Ubuntu 16 and uses these packages:

- tensorflow-gpu>=1.13
- absl-py
- atari-py
- gin-config
- opencv-python
- gym
- numpy

Citing
------
If you find this open source release useful, please reference in your paper:

> Agarwal, R., Schuurmans, D. & Norouzi, M.. (2019).
> An Optimistic Perspective on Offline Reinforcement Learning
> *arXiv preprint arXiv:1907.04543*.

    @article{agarwal2019striving,
      title={An Optimistic Perspective on Offline Reinforcement Learning},
      author={Agarwal, Rishabh and Schuurmans, Dale and Norouzi, Mohammad},
      journal={arXiv preprint arXiv:1907.04543},
      year={2019}
    }
    
Note: A previous version of this work was titled "Striving for Simplicity in Off
Policy Deep Reinforcement Learning" and was presented as a contributed talk at
NeurIPS 2019 DRL Workshop.

Disclaimer: This is not an official Google product.
