This is a demo of spark task creation and shuffle operations.

## Setting up the environment

For this demo you need:

- conda (can follow the steps [from here](https://docs.conda.io/projects/conda/en/latest/user-guide/install/index.html) to install it)

After installing conda, execute the below command to setup all required dependencies

```
conda env create -f environment.yml
```
This will create a new conda env named ```spark``` with all required dependencies installed.
Then, activate the env

```
conda activate spark
```

Now you are ready to follow along, fire up jupyter-lab and navigate to ```spark.ipynb``` notebook to start executing the experiment.

```
jupyter-lab
```