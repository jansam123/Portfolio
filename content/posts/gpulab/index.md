---
title: "Setup MFF UK GPULab"
date: 2022-09-18T08:06:25+06:00
# hero: /images/posts/writing-posts/analytics.svg
description: "Setup a tensorflow with CUDA and cuDNN on MFF UK GPULab."
theme: Toha
menu:
  sidebar:
    name: MFF UK GPULab
    identifier: gpulab
    weight: 2
hero: nvidia_smi.jpg
---

## Prerequisites
1. Get a CAS login.
2. Login to gpulab using CAS login, follow this [guide](https://gitlab.mff.cuni.cz/mff/hpc/clusters#installed-software-on-clusters). 


## Charliecloud image
3. Log into a gpu node
    ```bash
    srun -p gpu-ffa --gpus=1 --time=5:00:00 --pty bash
    ```
4. Pull nvidia docker image with Charliecloud
    ```bash
    ch-image pull tensorflow/tensorflow:latest-gpu
    ```
5. Convert the docker image to charliecloud image expressed as a directory `./my-tf` 
    ```bash
    ch-convert -i ch-image -o dir tensorflow/tensorflow:latest-gpu ./my-tf
    ```
6. Import CUDA libraries
    ```bash
    ch-fromhost --nvidia ./my-tf
    ``` 
7. Launch the container
   ```bash
    ch-run -w -c /home/jankovys --bind=/home/jankovys -u 0 -g 0 ./my-tf -- bash
    ``` 
The command above will launch the container with working directory `/home/jankovys` and write access to the container (`-w`). The `--bind=/home/jankovys` option will bind the `/home/jankovys` directory on the host to the `/home/jankovys` directory in the container. The `-u 0 -g 0` options will run the container as root user. The `--` at the end of the command tells ch-run that the command to run in the container follows. 

8. Verify the GPU support
    ```bash
    nvidia-smi
    ```

## [Tensorflow](https://www.tensorflow.org/install/pip)
If you want to install tensorflow, it must be **the same version as tensorflowe in the charliecloud image**.
16. Upgrade pip
```bash
pip install --upgrade pip
```
17. create a virtual environment
```bash
python -m venv venv
source venv/bin/activate
```
18. Install tensorflow
```bash
pip install tensorflow
```
19. Verify the installation and GPU support
    ```bash
    python -c "import tensorflow as tf; print(tf.reduce_sum(tf.random.normal([1000, 1000])))"
    ```
    You should see something like this:
    ```bash
    tf.Tensor(0.0, shape=(), dtype=float32)
    ```
    Verify GPU support
    ```bash
    python -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
    ```
    You should see something like this:
    ```bash
    [PhysicalDevice(name='/physical_device:GPU:0', device_type='GPU')]
    ```

## Running scripts
To run a python script `runMe.py` create a file `runMe.sh` with the following content:
```bash
#!/bin/bash
#SBATCH --partition=gpu-ffa                                  # partition you want to run job in
#SBATCH --gpus=1                                             # number of GPUs
#SBATCH --mem=16G                                            # CPU memory resource
#SBATCH --time=12:00:00					                     # time limit
#SBATCH --cpus-per-task=8                                    # cpus per tasks
#SBATCH --job-name="run_conda"                               # change to your job name
#SBATCH --output=/home/jankovys/JIDENN/out/%x.%A.%a.log      # output file    

ch-run -w --bind=/home/jankovys -c /home/jankovys/JIDENN /home/jankovys/my-tf -- bash_scripts/runner_inside.sh
```
and a file `runner_inside.sh` with the following content:
```bash
#!/bin/bash
venv/bin/python runMe.py
```
To import the CUDA libraries, you need start the sbatch job from a working node:
```bash
srun -p gpu-ffa --gpus=1 --pty bash
sbatch runMe.sh
```
The `runMe.sh` script will automatically log onto a working node with selected resources, launch the container, and run the `runner_inside.sh` script. 
The `runner_inside.sh` script will activate the virtual environment and run the `runMe.py` script with CUDA support.

To run the script in a interactive session, run the following command:
```bash
srun -p gpu-ffa --gpus=1 --time=5:00:00 --pty bash
ch-run -w -c /home/jankovys --bind=/home/jankovys -u 0 -g 0 ./my-tf -- bash
source venv/bin/activate
python runMe.py
```



## Troubleshooting
### `Could not load library libcudnn_cnn_infer.so.8. Error: libcuda.so: cannot open shared object file: No such file or directory` error
When using CNN in TF you might get the following error:
```bash
    Could not load library libcudnn_cnn_infer.so.8. Error: libcuda.so: cannot open shared object file: No such file or directory
```
To fix this create a link inside your charliecloud image:
```bash
    ln -s /usr/local/cuda/targets/x86_64-linux/lib/libcuda.so.1 /usr/local/cuda/targets/x86_64-linux/lib/libcuda.so
```


<!-- ## Notes
### Syntax highlighting and autocompletion in IDE
Include this lines in `__init__.py` file of tensorflow if you autocompletion and syntax highlighting in your IDE

    ```python
    # Explicitly import lazy-loaded modules to support autocompletion.
    # pylint: disable=g-import-not-at-top
    if _typing.TYPE_CHECKING:
        from tensorflow_estimator.python.estimator.api._v2 import estimator as estimator
        from keras.api._v2 import keras
        from keras.api._v2.keras import losses
        from keras.api._v2.keras import metrics
        from keras.api._v2.keras import optimizers
        from keras.api._v2.keras import initializers
    # pylint: enable=g-import-not-at-top
    ```
### Error when using CNN in TF
When using CNN in TF you might get the following error:
```bash
    Could not load library libcudnn_cnn_infer.so.8. Error: libcuda.so: cannot open shared object file: No such file or directory
```
To fix this, you need to add the following lines to `~/.bashrc` (use `source ~/.bashrc` to apply changes):
```bash
    export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
    export LD_LIBRARY_PATH=/usr/local/cuda/targets/x86_64-linux/lib/:$LD_LIBRARY_PATH
```
And create symlink for `libcuda.so`:
```bash
    ln -s /usr/local/cuda/targets/x86_64-linux/lib/libcuda.so.1 /usr/local/cuda/targets/x86_64-linux/lib/libcuda.so -->
```