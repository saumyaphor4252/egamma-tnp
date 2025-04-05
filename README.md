# egamma-tnp
E/Gamma High Level Trigger efficiency from NanoAOD using Tag and Probe in the [coffea](https://github.com/CoffeaTeam/coffea) framework.

## Initial Setup
To use jupyter notebooks on a cluster (LPC for instance), choose a 3-digit number to replace the three instances of `xxx` below. The setup is similar for LXPLUS. We also port-forward 8787 to monitor the dask dashboard.
```bash
# connect to LPC with a port forward to access the jupyter notebook server and the dask dashboard
# remember to `kinit USERNAME@FNAL.GOV` to set up kerberos authorization before logging in
ssh USERNAME@cmslpc-sl7.fnal.gov -L8xxx:localhost:8xxx -L8787:localhost:8787

# or 

ssh USERNAME@lxplus.cern.ch -L8787:localhost:8787
```

Then create a working directory, clone the repository and enter the directory
```bash
cd /afs/cern.ch/work/s/ssaumya/private/Egamma/IasonTool/
git clone -b 2024_Studies git@github.com:saumyaphor4252/egamma-tnp.git
cd egamma-tnp/
```
The package can in principle be installed in any python virtual environment with python 3.8 or higher. 
That means that you can use it within a conda environment, a Singularity or Docker container, an LCG release, or any other python environment without any conflicting packages.

```bash
python3 -m venv egmtnpenv
source egmtnpenv/bin/activate
pip install jupyter
pip install ipython
pip install ipykernel
ipython kernel install --user --name=egmtnpenv
python -m ipykernel install --user --name=egmtnpenv
pip install . --no-cache-dir
```

Proxy might be needed before depending on the particular job/task.
```
voms-proxy-init --voms cms --valid 100:00
```

In general, you don't have to use jupyter to run this package however, there are a lot of convenient features available in jupyter notebook that make this tool very expressive. 
Jupyter comes pre-installed in the singularity images. 
If you want to use any other type of environment and want to use jupyter notebooks, make sure it has jupyter (`lab`, `notebook` or `nbclassic`) installed.

To start the jupyter notebook, do
```bash
jupyter lab --no-browser --port 8xxx
```

There should be a link like `http://localhost:8xxx/?token=...` displayed in the output at this point, paste that into your browser.
You should see a jupyter notebook with a directory listing.

### Next time you want to run

```
ssh -o ServerAliveInterval=10 ssaumya@lxplus9.cern.ch -L8777:localhost:8777
cd /afs/cern.ch/user/s/ssaumya/Egamma/egamma-tnp/
source egmtnpenv/bin/activate
jupyter lab --no-browser --port 8777

# Notebook template: https://github.com/saumyaphor4252/egamma-tnp/blob/2024_Studies/SpikeKiller_FineTuning.ipynb
```

## Running the code
This is the basic idea of how to run the code.
The instructions below may be outdated so please refer to the docstrings of the classes and functions for more information
and contact the author for any questions.

First you define the `Client` that you want to use. For the default dask `Client` you would do
```python
from distributed import Client
client = Client()
```

More basic examples of dask client usage can be found [here](https://distributed.dask.org/en/latest/client.html).

After that you need to define a fileset to calculate efficiencies over. 
In coffea, a fileset is a dictionary of the form
```python
fileset = {
    "ZJets": {
        "files": {
            "path/to/file1.root": "Events",
            "path/to/file2.root": "Events",
        }
    },
    "DiPhoton": {
        "files": {
            "path/to/file3.root": "Events",
            "path/to/file4.root": "Events",
        }
    },
}
```

This contains all the different datasets to run over. 
The keys are the dataset names and the values are dictionaries with the files and the tree names.
In this example we used local paths but the file paths can also be remote paths like xrootd or http.
To construct a fileset, you can query DAS through `rucio` using `coffea`'s dataset tools. Documentation on how to do that can be found in [this notebook](https://github.com/CoffeaTeam/coffea/blob/master/binder/dataset_discovery.ipynb).
This notebook also explains how to preprocess the fileset you queried for to remove any unavailable files using the `DataDiscoveryCLI` tool either from the command line or through python.
You can also preprocess a fileset with the preprocessing function itself
```python
from coffea.dataset_tools import preprocess

fileset_available, fileset_updated = preprocess(
    fileset, skip_bad_files=True
)
```
Now you want to use the available fileset to perform your tag and probe on. Suppose you want to do it for the `HLT_Ele30_WPTight_Gsf` trigger.
You would use the `ElePt_WPTight_Gsf` class as follows
```python
from egamma_tnp.triggers import ElePt_WPTight_Gsf

tag_n_probe = ElePt_WPTight_Gsf(
    fileset_available,
    30,
    goldenjson="json/Cert_Collisions2023_366442_370790_Golden.json",
)
```

Please refer to its docstring for more information on the arguments.
Then to perform tag and probe to get the $P_T$, $\eta$ and $\phi$ histograms of the passing and all probes

```python
histograms, report = tag_n_probe.get_tnp_histograms(
    uproot_options={"allow_read_errors_with_report": True},
    plateau_cut = 35,
    eta_regions_pt={
        "barrel": [0.0, 1.4442],
        "endcap": [1.566, 2.5],
    },
    compute=True,
)
```

Both `histograms` and `report` are dictionaries that have the datasets of the fileset as keys and the values are dictionaries that contain all the requested histograms and awkward arrays that contain the reports about file access errors respectively.
Suppose we used as `fileset_available` the fileset that we defined previously and you want to plot the efficiencies for the `ZJets` dataset as a function of $P_T$, $\eta$ and $\phi$.
To do this you would do
```python
from egamma_tnp.plot import plot_efficiency

hpt_pass_barrel, hpt_all_barrel = histograms["ZJets"]["pt"]["barrel"].values()
hpt_pass_endcap, hpt_all_endcap = histograms["ZJets"]["pt"]["endcap"].values()
heta_pass, heta_all = histograms["ZJets"]["eta"]["entire"].values()
hphi_pass, hphi_all = histograms["ZJets"]["phi"]["entire"].values()

plot_efficiency(hpt_pass_barrel, hpt_all_barrel)
plt.show()
plot_efficiency(hpt_pass_endcap, hpt_all_endcap)
plt.show()
plot_efficiency(heta_pass, heta_all)
plt.show()
plot_efficiency(hphi_pass, hphi_all)
plt.show()
```
The customization of those plots is up to the user. Using [mplhep](https://github.com/scikit-hep/mplhep) is recommended.
