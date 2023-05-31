# Running **Jupyter** on *galahad* in your local browser

This is a tutorial aiming to guide you through setting up your own `conda` environment on *galahad*, and launch **Jupyter** sessions through `slurm`. It is aimed to be as detailed as possible, but if you already have experiences with linux, conda, bash, slurm etc you should probably just [read this](https://nero-docs.stanford.edu/jupyter-slurm.html) instead.

## Installing `conda` locally
`conda` allows you to create your own computing environment with its separate python installation and packages. It will come in handy for running **Jupyter** and `casa`. You want to make sure that your `conda` installation is **local**, and installed in your `/share/nas/$USER` (or another nas such as `/share/nas-0-3/team_folder/your_folder/` for SKA data challenge) and **not in your home directory**!

[Download miniconda](https://docs.conda.io/en/latest/miniconda.html): In your command line on galahad, do
```console
cd /share/nas/$USER
```
or whereever your nas dir is. and then
``` console
#download miniconda
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
#install
bash Miniconda3-latest-Linux-x86_64.sh
```
Once you accept the license terms (by typing yes), you will be asked about the installation directory. **Make sure you don't press ENTER!** You can see something like
```
Miniconda3 will now be installed into this location:
/home/zchen/miniconda3

  - Press ENTER to confirm the location
  - Press CTRL-C to abort the installation
  - Or specify a different location below

[/home/zchen/miniconda3] >>>
```
You don't want that! Instead, specify a different location, something like `/share/nas/$USER/miniconda3`.

After installation, run `conda init` or simply do `source ~/.bashrc`. You will see that your bash kernel changes with a `(base)` prefix added, indicating that conda has been installed and now you are at the base environment.

## Setting up `conda` environment
Now let's create our own `conda` environment. For the purpose of this tutorial let's set up an environment for dealing with the SKA science data challenge stuff. Naively, we want the basic packages (`numpy`,`matplotlib`,`astropy`, ...) as well as things like **Jupyter** and `casa`. In your terminal type in
```console
conda create -n skasdc python=3.8
```
here `skasdc` is the name of the environment (you can name it anything you want), and I specify the python version to be 3.8 (and you can choose your favorite). Then you can activate the environment and install the packages
```console
conda activate skasdc
pip install jupyterlab --no-cache-dir
```
here, `--no-cache-dir` tells pip to not cache any data, which is quite useful because it is usually cached at your home directory and you can potentially exceed your disc quota quite quickly.

`Jupyter lab` has a few very useful extensions that I recommend installing as well:
```console
# this tells you how much resoure you are using
pip install jupyter-resource-usage --no-cache-dir
# this gives a GUI for git workflow
pip install --upgrade jupyterlab jupyterlab-git --no-cache-dir
```
Installing a bunch of other useful packages including `casa`:
```console
pip install scipy matplotlib mpi4py --no-cache-dir
pip install casatools casatasks casaplotms casaviewer --no-cache-dir
```

## Running `jupyter lab` on a slurm node
Galahad is a HPC that uses [slurm](https://slurm.schedmd.com/documentation.html) to manage the workload. If you want, read [this intro](https://blog.ronin.cloud/slurm-intro/) to get a basic idea of how to use it. In short, you should submit a bash script to tell slurm how many resources you need and what you want to do, and it schedules your job to be run on some node(s).

The head (log-in) node on galahad should only be used to do basic command line stuff and not for anything computational. Usually, you would submit a slurm bash script for your job to be run on some computing nodes. In order to do that for jupyter, you need to first launch jupyter through slurm on a specific **port**, and then use **ssh port forwarding** to link a port number on your local machine to the jupyter port on galahad.

On galahad, in the directory where you want to launch jupyter, create a `runlab.sh` (or whatever name you prefer) file and write
```bash
#!/bin/bash
#SBATCH --time=10:00:00
#SBATCH --mem=63G
#SBATCH --job-name=jupyter_lab
#SBATCH --constraint="A100"


echo ">>> activate conda"
source /home/zchen/.bashrc
source activate skasdc
cat /etc/hosts

echo ">>> run jupyter"
jupyter lab --ip=0.0.0.0 --no-browser
```
here all the `SBATCH` lines can be changed according to your needs and read [the docs](https://slurm.schedmd.com/sbatch.html) before you change things. `--constraint="A100"` tells slurm to use a GPU node so delete it if you don't need to.

Alternatively, maybe you can just use a specific node. In that case, the start of the sbatch file can be
```
#SBATCH --time=10:00:00
#SBATCH --job-name=jupyter_lab
#SBATCH -w compute-0-x
```
here `compute-0-x` is the name of the node. You can check which nodes are idle by entering `sinfo` into the terminal.


Then, run `sbatch runlab.sh` and check if your job is running by entering `squeue --user=$USER`. You should see something like
```console
JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
80616   CLUSTER  jupyter    zchen  R       0:36      1 compute-0-119
```
and there will be a `slurm-*.out` file (in this case `slurm-80616.out`). vim it and you should see the ip address for the node and the port number
```txt
>>> start
127.0.0.1       localhost.localdomain localhost
130.88.9.226    galahad
192.168.1.181   compute-0-119.local compute-0-119
>>> run jupyter
...
To access the server, open this file in a browser:
file:///home/zchen/.local/share/jupyter/runtime/jpserver-233351-open.html
Or copy and paste one of these URLs:
http://compute-0-119.local:8888/lab?token=xxx
http://127.0.0.1:8888/lab?token=xxx
```
I changed the `token=xxx` part, in reality you should see a very long token. You can see that in this case the jupyter lab is launched on node `compute-0-119` with ip address `192.168.1.181` and on port 8888.

Now, start another terminal on your own computer and add a bash function (if you are on linux; window and mac should have similar things) to your `~/.bashrc`
```bash
# change the last part to your own username
sshextport() {
    ssh -Llocalhost:$1:localhost:$2 zchen@external.jb.man.ac.uk
}
```
then run `source ~/.bashrc` (you only need to add the bash function and source .bashrc once). Then you can link your own local port to a port on external by logging onto external
```console
sshextport 41234 41235
```
here, `41234` is the port on your own computer whereas `41235` is the port on external. They don't need to be different numbers, here I am just choosing different ones for illustration. The default port number for **Jupyter** is `8888`, and any number below 65535 can do as long as you avoid the [commonly used ones](https://www.cloudflare.com/en-gb/learning/network-layer/what-is-a-computer-port/). **DO NOT just copy the number above. Choose your own random port number to avoid forwarding issue**.

Now you should be on external. The next step is to link the `41235` to wherever jupyter lab is run on galahad, in this case ip address `192.168.1.181` and port `8888`. Add a bash function to your `~/.bashrc` on external
```bash
# change the last part to your own username
sshgalahadip() {
    ssh -L$1:$2:$3 zchen@galahad.ast.man.ac.uk
}
```
and run
```console
bash
source ~/.bashrc
```
Again, you only need to add the bash function and `source .bashrc` once. Then you can link the external to galahad by entering
```console
sshgalahadip 41235 192.168.1.181 8888
```
here, 41235 is the port number on external, 192.168.1.181 is the ip address for the node, 8888 is the port number on galahad where jupyter is being run.

Finally in your own browser (not some vnc, just your own browser on your own computer), go to http://localhost:41234/lab?token=xxx
Remeber, `41234` is something you choose when entering `sshextport 41234 41235`, and the `token=xxx` is something you should copy and paste from the `slurm` output file. You should be in!


## Some useful stuff
- Upload/download **small** files: if you have a jupyter session on, simply drag the file to the file directory on the left hand side of the jupyter page. Right-click the files and click "download" to your own computer.
- If you want **slightly bigger** file transferred or haven't sorted out jupyter you can do `scp -o "ProxyJump zchen@external.jb.man.ac.uk" zchen@galahad.ast.man.ac.uk:/share/nas/zchen/runlab.sh runlab.sh`. Replace the file and user name to your own of course. In short, `ProxyJump` does the first part of ssh into external so you can then copy files on galahad through external to your machine, or the other way around.
- For large files you should use [globus](https://www.globus.org/data-transfer).


- If you want more detailed information on the slurm nodes you can specify `sinfo` output format. [Read the docs](https://slurm.schedmd.com/sinfo.html) or just copy this:
```console
sinfo -o "%20n %20f %20m %20c %20t"
```
This usually is enough.