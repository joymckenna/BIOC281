# Tutorial 3
Today we will be working through two notebooks, one written in R and the other python, that use the Seurat (https://satijalab.org/seurat/) and scanpy (https://scanpy.readthedocs.io/en/stable/) packages to cluster part of the human lung cell atlas single cell dataset (https://hlca.ds.czbiohub.org). The notebooks are designed to build intuition for how changing various paramters affects clustering results and illustrate the utility of iterative subclustering. In addition to the standard normalization, the notebooks also demo some of the more complex procedures we discussed during the lecture as well as methods for integrating datasets constructed with different technologies.

To begin the tutorials, we need to request dedicated resources on Stanford FarmShare, launch a jupyer notebbook, then download the data from Synapse.

## Login to FarmShare
```bash
ssh <SUNetID>@rice.stanford.edu
```

### Switch to the persistent terminal window
The resources we requested on Wednesday should be ready to go. You can tell if your terminal has switched from something like "(base) <SUNetID>@**rice**<XX>:~$" to "(base) <SUNetID>@**wheat**<XX>:~$"

You should also see the green tmux bar along the bottom of your terminal.
```bash
tmux attach
```
If the resources have been allocated, then proceed. If not, please let us know!

## Activate the single cell environment
```bash
source activate singlecell
```

## Start jupyter
```bash
jupyter lab --no-browser
```

## Create an SSH tunnel (on your system)
```bash
ssh -N -f -L <Port>:localhost:<Port> <SUNetID>@wheat<XX>.stanford.edu
```

## Login to jupyter in your browser, open a Terminal window, and activate the single cell environment
```bash
source activate singlecell
```

## Pull the GitHub repository
```bash
git pull
```

## Navigate to the current tutorial folder
```bash
cd ~/BIOC281/Classes/3
```

## Login to synapse
```bash
synapse login -u <Username> --remember-me
```

## Download Human Lung Cell Atlas data and metadata from synapse (Travaglini et al (2020) _Nature_)
This dataset contains ~75,000 cells captured using 10x (90%) and SmartSeq2 (10%) from the lung of 3 human patients.

```bash
synapse get syn21560510

synapse get syn21560511

synapse get syn21560409

synapse get syn21560410
```

## Open and complete BIOC281/Classes/3/Tutorial 3 - Seurat.ipypnb
When finished, please save the outputs of Tutorial 3 - Seurat.ipynb (File > Export Notebook As... > Export Notebook to HTML) when you have completed it and upload it to Canvas.

## Open and complete BIOC281/Classes/3/Tutorial 3 - scanpy.ipynb
When finished, please save the outputs of Tutorial 3 - scanpy.ipynb (File > Export Notebook As... > Export Notebook to HTML) when you have completed it and upload it to Canvas.

## Request resources for the next class
After saving Tutorial 3 - Seurat.ipynb and Tutorial 3 - scanpy.ipynb, return to the original terminal window where you launched jupyter. Quit jupyter by pressing command+c twice.

### Leave the current wheat allocation
After running the command below, your terminal should switch back to rice from wheat.
```bash
exit
```

### Run salloc
You should see the green tmux bar
```bash
salloc --ntasks-per-node=1 --cpus-per-task=4 --mem=30G --time=0-3:00:00 --begin="13:30:00 10/26/20" --qos=interactive srun --pty bash -i -l
```
    
You can now detach from tmux with control+b and then press "d". If the green bar on the bottom disappears, you can safely close your terminal window. See you next class!