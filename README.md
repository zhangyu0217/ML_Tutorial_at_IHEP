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
#setup your own ROOT. I recommend ROOT5.
```
Each time your want to run the code, you should activate the environment.
```shell
#de-activate the env
deactivate

```

## Install needed packages
```shell
pip install sklearn
pip install xgboost
pip install uproot
pip install matplotlib<2.0
```
Copy my package to your directory
```shell
cp /publicfs/atlas/atlasnew/higgs/hgg/guofy/ML_from_Yu_Zhang/Tutorial/WorkDir $YOUR_DIRECTORY

```

# Config
```python
config={
'signal_file_list': ['/publicfs/atlas/atlasnew/higgs/hgg/guofy/ML_from_Yu_Zhang/Tutorial/WorkDir/data/samples/signal/hhto4l.root'], 
'bkg_file_list': [	'/publicfs/atlas/atlasnew/higgs/hgg/guofy/ML_from_Yu_Zhang/Tutorial/WorkDir/data/samples/background/VVV.root',
		 	'/publicfs/atlas/atlasnew/higgs/hgg/guofy/ML_from_Yu_Zhang/Tutorial/WorkDir/data/samples/background/qqZZ.root',
			'/publicfs/atlas/atlasnew/higgs/hgg/guofy/ML_from_Yu_Zhang/Tutorial/WorkDir/data/samples/background/ttZ.root',
			'/publicfs/atlas/atlasnew/higgs/hgg/guofy/ML_from_Yu_Zhang/Tutorial/WorkDir/data/samples/background/ttbar.root'], 
'signal_tree_name': 'quadlep', 
'methods': ['BDTG'], 
'bkg_selection': '', 
'WorkDir': '/publicfs/atlas/atlasnew/higgs/hgg/guofy/ML_from_Yu_Zhang/Tutorial/WorkDir/', 
'bkg_tree_name': 'quadlep', 
'signal_weight_name': 'weight', 
'dump_signal_variables': ['met', 'met_phi', 'metsumet', 'metsig', 'mZ1', 'mZ2', 'm4l', 'DphiZ1', 'DphiZ2', 'ptl1', 'ptl2', 'ptl3', 'ptl4', 'pt4l', 'ptZ1', 'ptZ2', 'Njets', 'Nbjets77'], 
'dump_bkg_variables': ['met', 'met_phi', 'metsumet', 'metsig', 'mZ1', 'mZ2', 'm4l', 'DphiZ1', 'DphiZ2', 'ptl1', 'ptl2', 'ptl3', 'ptl4', 'pt4l', 'ptZ1', 'ptZ2', 'Njets', 'Nbjets77'], 
'variables_name': ['MET', 'MET_{#Phi}', '#sum E_{T}', 'MET_{sig}', 'm_{Z1}', 'm_{Z2}', 'm_{4l}', '#Delta#Phi_{Z1}', '#Delta#Phi_{Z2}', 'p_{T#l 1}', 'p_{T#l 2}', 'p_{T#1 3}', 'p_{T#l 4}', 'p_{T4l}', 'p_{TZ1}', 'p_{TZ2}', 'N_{jets}', 'N_{bjets77}'], 
'signal_selection': '', 
'bkg_weight_name': 'weight', 
'test_size': 0.5, 
'outputDir': 'test'
}
```
The config is written as a dicationary in python/config.py and imported in the script.
This is a little stupid, but it seems py2 could load json file correctly due to de-coding issue.

Currently, only methods of "BDTA, BDTG, XGBoost" are available.
```python
import config
config=config.config
```

# Function

## Import module
```python
#python module
import numpy as np
import json
import logging
import time
import matplotlib.pyplot as plt
import os

#sklearn module
import sklearn
from sklearn import tree
from sklearn.tree import DecisionTreeClassifier, DecisionTreeRegressor
from sklearn.ensemble import AdaBoostRegressor, AdaBoostClassifier
from sklearn.ensemble import GradientBoostingRegressor
import xgboost as xgb
from xgboost import plot_tree
from sklearn.metrics import mean_squared_error
from sklearn.metrics import f1_score,precision_score,recall_score,roc_auc_score,accuracy_score,roc_curve
from sklearn.model_selection import GridSearchCV,validation_curve
import joblib

#ROOT module
import ROOT
from ROOT import *
import uproot

```
## Import data
```python
def importData_uproot(config):
   signal_dataset, signal_weight = importFromFileList(config["signal_file_list"],config["signal_tree_name"],config["dump_signal_variables"],config["signal_weight_name"])
   bkg_dataset, bkg_weight = importFromFileList(config["bkg_file_list"],config["bkg_tree_name"],config["dump_bkg_variables"],config["bkg_weight_name"])
   print("shape of signal dataset : ")
   print(np.shape(signal_dataset))
   print(np.shape(bkg_dataset))
   print(np.shape(signal_weight))
   print(np.shape(bkg_weight))
   return signal_dataset,signal_weight,bkg_dataset,bkg_weight

def importFromFileList(file_list, tree_name, branch_name, weight_name):
   dataset =[]
   w = []
   for i in range(len(file_list)):
      dataset_tmp = []
      t = uproot.open(file_list[i])[tree_name]
      dict_v = t.arrays(branch_name)
      dict_w = t.array(weight_name)
      for j in range(len(branch_name)):
         dataset_tmp.append(dict_v[branch_name[j]])
      dataset_tmp=np.array(dataset_tmp)
      dataset_tmp=dataset_tmp.transpose().tolist()
      dataset.extend(dataset_tmp)
      w.extend(dict_w)
      #!!!apply the selelction!!!
      for event in range(len(dataset)-1,-1,-1):
         if w[event]>0.1 or w[event]<0:
            dataset.pop(event)
            w.pop(event)
   dataset = np.array(dataset)
   sumOfWeight = np.sum(w)
   w=[1*i/sumOfWeight for i in w]
   w = np.array(w)
   return dataset, w

```

The function "importFromFileList" returns the array of dataset and weights.

The selection is hard-coding here. I will optimize it in the future.
## Plot variables
Plot the distribution of variables and the correlation matrix. This is pure ROOT-based plotting code.
## Training and Test
pure ROOT-based code
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
The ranking is evaluated by the gain on loss function from individual variable.

The gain is divided by the total gain.
## Hyper-parameter optimization
GridSearchCV

RandomizedSearchCV

# Execute
```shell
python python/classification_ML_lxslc.py
```

# Output
```shell
$outputDir/
├ data
├ log
├ model
├ plot
└ root
```
data : dataset saved in .json as array

log : log info

model : saved model of classifier

plot : plots of distrubtion/correlation/overtraing/importance/ROC

root : root file of histograms/plots
# To-do
```diff
- the error bar in overtraining plot is correct?
! BDTG is quite slow when n_estimators is serveral hundred
```
+ draw the weighted correlation matrix
+ hyper-parameter optimization
+ add more methods like RNN, DNN
# Issues
```diff
+ move to py3 to deal with the weight correctly
+ optimize the plotting code in "plotOvertraining"
+ check the performance BDTG vs XGBoost by tunning the hyper-parameters
```
