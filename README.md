# Procgen OOD Benchmark


ProcgenOOD is an extension of the standard [Procgen benchmark environment](https://github.com/openai/procgen) that evaluates on configurable level generation variables out-of-distribution. These predefined variables are called *holdout types* and correspond to random generation of an asset or value within the Procgen games. See [[#Holdout Types]] below for specific types and game support for each. 

Like the original be, ProcgenOOD contains 16 procedurally generated [gym](https://github.com/openai/gym) environments that run fast. 

This README describes our extension and changes to the original repo, along with basic installation and usage information to fit most users' needs. For more information on original game descriptions and issues, 

> [!todo] 
> The associated training repository containing the (future) paper's experimental results has yet to be open-sourced. 


<img src="https://raw.githubusercontent.com/openai/procgen/master/screenshots/procgen.gif">


### Why is this extension needed? 

In the Procgen benchmark, algorithms are trained on a fixed number of level seeds (typically 500) and then evaluated by uniformly sampling level seeds from $2^{32} - 1$ (integer max). Meanwhile, each of the levels sample these holdout types independently. This means that: 
1. Evaluation levels are sampled independently (IID) from the same distribution as training levels, and 
2. There is no direct way to control the level generation to test algorithms' specific capabilities. 

Approaches that help RL performance IID often do not transfer to OOD, even by small distribution shifts. As RL algorithms are inherently learning with non-stationary targets and typically deployed with sim2real transfer, we believe that evaluating OOD is far more valuable to the research community. 


<br>

## Installation 

The following instructions assume you have `conda` installed. 
If you do not have `conda`, you can install it from [Miniconda](https://docs.conda.io/en/latest/miniconda.html).



```bash
cd /path/to/your/clone/of/procgen-ood 
conda create -n procgen_ood -f environment.yml
conda activate procgen_ood
pip install -e .  # install the package in editable mode 
``` 

Verify the installation by running the following command:

```bash 
python -m procgen.interactive --env-name coinrun 
```



<br>

## Holdout Types 

The supported holdout types during training and/or evaluation are `all`, `background`, `enemy`, `platform`, and `none` .  In addition to holdout types, a holdout fraction value is supported to provide finer control over the amount of data held out. These are further specified with holding out during training or evaluation. The predefined holdout types are: 

* `all` - Hold out all supported types (see following table). 
* `background` - Hold out background images. 
* `enemy` - Hold out enemy sprites. 
* `platform` - Hold out platforming level difficulties. 
* `none` - Disable hold out. Along with `--num-levels=500`, this replicates the original Procgen benchmark. 

| \#  | Game \\ Supports Holdout Type | agent | enemy | platform | background | all |
| --- | ----------------------------- | ----- | ----- | -------- | ---------- | --- |
| 1   | bigfish                       |       | ✔     |          | ✔          | ✔   |
| 2   | bossfight                     | ✔     | ✔     |          | ✔          | ✔   |
| 3   | caveflyer                     |       |       |          | ✔          | ✔   |
| 4   | climber                       | ✔     |       | ✔        | ✔          | ✔   |
| 5   | coinrun                       | ✔     | ✔     | ✔        | ✔          | ✔   |
| 6   | dodgeball                     |       | ✔     |          | ✔          | ✔   |
| 7   | fruitbot                      |       |       |          | ✔          | ✔   |
| 8   | heist                         |       |       | ✔        | ✔          | ✔   |
| 9   | jumper                        |       |       |          | ✔          | ✔   |
| 10  | leaper                        |       |       | ✔        | ✔          | ✔   |
| 11  | maze                          |       |       |          | ✔          | ✔   |
| 12  | miner                         |       |       |          | ✔          | ✔   |
| 13  | ninja                         |       |       | ✔        | ✔          | ✔   |
| 14  | starpilot                     |       |       |          | ✔          | ✔   |
|     | ~~chaser~~                    |       |       |          |            |     |
|     | ~~plunder~~                   |       |       |          |            |     |

> [!NOTE]  
> The behavior of holdout type "all" is **game specific!** Holdout type "all" independently samples all other supported types. 
> - E.g., `coinrun` with holdout type "all" will independently sample each of \["background", "agent", "enemy", "platform"\] variables using the accompanying `--[train/eval]-holdout-frac 0.1` argument. 
  > In contrast, `bigfish` only supports randomizing over "enemy" & "background".
> - As seen in the table above, `chaser` and `plunder` do not support any holdout types. 


<br>

## Environment Options

These options from the base environment are relevant for testing OOD with this codebase: 

* `env_name` - Name of environment, or comma-separate list of environment names to instantiate as each env in the VecEnv.
* `num_levels=0` - The number of unique levels that can be generated. Set to 0 to use unlimited levels.
* `start_level=0` - The lowest seed that will be used to generated levels. 'start_level' and 'num_levels' fully specify the set of possible levels.
* `debug=False` - Set to `True` to use the debug build if building from source.
* `debug_mode=0` - A useful flag that's passed through to procgen envs. Use however you want during debugging.

The following options are new pertaining to OOD holdout types: 

- `eval_holdout_type="none"` - Predefined type to hold out during **evaluation**. 
- `train_holdout_type=None` - Predefined type to hold out during **training**. 
- `eval_holdout_frac=0.0` - During **evaluation**, withhold this fraction of the specified holdout type.  Value must be in \[0,1\]. 
- `train_holdout_frac=None` - During **training**, withhold this fraction of the specified holdout type.  Value must be in \[0,1\]. 
- `holdout_sampling_mode="extrapolate"` - Withholding ranges from well-ordered random variables can either be at the high end ("extrapolate", default) or somewhere in the middle ("interpolate"). There is no distinction for categorical variables (e.g., background assets). 


Here's how to set the options:

```python
import gym
env = gym.make(
	"procgen:procgen-coinrun-v0", 
	num_levels=0, # use all level seeds 
	train_holdout_type="background", 
	train_holdout_frac=0.5,   # hold out ~50% of backgrounds during training
	eval_holdout_type="none", # eval on full level distribution 
	eval_holdout_frac=0.0,
	# eval_holdout_frac=0.5,  # equivalent: with type "none", frac is ignored 
)
```

> [!NOTE] NOTE: Since the gym environment is adapted from a gym3 environment, early calls to `reset()` are disallowed and the `render()` method does not do anything.  
> - To render the environment, pass `render_mode="human"` to the constructor, which will send `render_mode="rgb_array"` to the environment constructor and wrap it in a `gym3.ViewerWrapper`.  
> - If you just want the frames instead of the window, pass `render_mode="rgb_array"`.


<br>

# License 

This project contains two different licenses for different parts of the code:

- The original code, which was forked from [Procgen](https://github.com/openai/procgen/tree/5e1dbf341d291eff40d1f9e0c0a0d5003643aebf), is licensed under the MIT license. You can find the MIT license in the `LICENSE-MIT` file.
- All modifications and additions made by Kevin Corder, Song Park, DEVCOM Army Research Laboratory, and/or Parsons Corporation are licensed under the CC0 1.0 Universal license. See the `LICENSE-CC0` file for details.


<br> 

# Citation

To cite this project in your work, please use the following Bibtex: 

(INSERT CITATION HERE)
