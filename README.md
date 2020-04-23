# ML_Tutorial_at_IHEP
This tutorial runs on IHEP server lxslc.ihep.ac.cn

# [Environment](#Environment)
# [Config](#Config)
# [Function](#Function)
# [Execute](#Execute)
# [Output](#Output)
# [To do](#To-do)

# Environment
"Python+ Scikit-learn+ PyROOT" is needed.

ROOT can not be imported correctly in Py3. This is a known long-term [issue](https://root-forum.cern.ch/t/pyroot-import-error-pyinit-libpyroot/16263/8). Let's build a virtual enrironment based on py2 to solve this.

## Build the virtual environment
```shell
#set up the py3
export PATH=$ROOTSYS/bin:/publicfs/atlas/atlasnew/higgs/hgg/guofy/ML_from_Yu_Zhang/Python-3.6.10/bin:$PATH
#setup the virtual environment with py2.7
mkdir virtualenv
cd virtualenv
virtualenv py2.7 -p /usr/bin/python2.7
#activate the virtual env
source py2.7/bin/activate
```
```shell
#de-activate the env
deactivate

```

## Install needed packages
```shell
pip install sklearn
pip install xgboost
pip install root_numpy
```

# Config

# Function

## Import module
## Import data
## Plot variables
## Training and Test
## Ranking
```python
def plotImportance(clf, size, outputDir,method):
    importances = clf.feature_importances_
    indices = np.argsort(importances)[::-1]

    # Print the feature ranking
    logging.info("Feature ranking:")

    for f in range(size):
            logging.info("%d. feature %d (%f)" % (f + 1, indices[f], importances[indices[f]]))

            # Plot the feature importances of the forest
    plt.figure()
    plt.title("Feature importances")
    plt.bar(range(size), importances[indices],
            color="r", align="center")
    plt.xticks(range(size), indices)
    plt.xlim([-1, size])
    plt.savefig(outputDir+"Importance_"+method+".png")
```
## Hyper-parameter optimization


# Execute
```shell
python python/classification_ML_lxslc.py
```

# Output
# To-do
