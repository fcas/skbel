<img src="/docs/img/illu-01.png">

[![PyPI version](https://badge.fury.io/py/skbel.svg)](https://badge.fury.io/py/skbel)
[![Build Status](https://travis-ci.com/robinthibaut/skbel.svg?branch=master)](https://travis-ci.com/robinthibaut/skbel)
[![Documentation Status](https://readthedocs.org/projects/skbel/badge/?version=latest)](https://skbel.readthedocs.io/en/latest/?badge=latest)
[![DOI](https://zenodo.org/badge/369214956.svg)](https://zenodo.org/badge/latestdoi/369214956)
[![Downloads](https://pepy.tech/badge/skbel)](https://pepy.tech/project/skbel)

```pip install skbel```

SKBEL is a Python package built on top of scikit-learn to implement the Bayesian Evidential Learning framework as described in Scheidt et al. (2018).

Documentation: [skbel.readthedocs.io](https://skbel.readthedocs.io/en/latest/)

Bayesian Evidential Learning - A Prediction-Focused Approach
-----------------------------------------------------------------------------------------
### Introduction

<p align="center">
<img src="/docs/img/evidential.png">
</p>
<p align="center">
  Figure 1: The concept of BEL. d = predictor (observed data), h = target (parameter of interest), m = model.
<p align="center">

- The idea of BEL is to find a direct relationship between `d` (predictor) and `h` (target) in a reduced dimensional space with machine learning.
- Both `d` and `h` are generated by forward modelling from the same set of prior models `m`.
- Given a new measured predictor `d*`, this relationship is used to infer the posterior probability distribution of the target, without the need for a computationally expensive inversion. 
- The posterior distribution of the target is then sampled and backtransformed from the reduced dimensional space to the original space to predict posterior realizations of `h` given `d*`.
  
### Workflow

#### Forward modeling
- Examples of both `d` and `h` are generated through forward modeling from the same model `m`. Target and predictor are real, multi-dimensional random variables.
#### Pre-processing
- Specific pre-processing is applied to the data if necessary (such as scaling).
#### Dimensionality reduction
- Principal Component Analysis (PCA) is applied to both target and predictor to aggregate the correlated variables into a few independent Principal Components (PC’s).
#### Learning
- Canonical Correlation Analysis (CCA) transforms the two sets into pairs of Canonical Variates (CV’s) independent of each other.
#### Post-processing
- Specific post-processing is applied to the CV's if necessary (such as CV normalization).
#### Posterior distribution inference
- The mean `μ` and covariance `Σ` of the posterior distribution of an unknown target given an observed `d*` can be directly estimated from the CV's distribution.
- Alternatively, the posterior conditional distribution can be inferred through KDE.
#### Sampling and back-transformation to the original space
- The posterior distribution is sampled to obtain realizations of `h` in canonical space, successively back-transformed to the original space.

<p align="center">
<img src="/docs/img/flow-01.png">
</p>
<p align="center">
  Figure 2: Typical BEL workflow.
<p align="center">
 
Example
-----------------------------------------------------------------------------------------
- All the details about the example can be found in [arXiv:2105.05539](https://arxiv.org/abs/2105.05539), and the code in `skbel/examples/demo.py`.
- It concerns a hydrogeological experiment consisting of predicting the wellhead protection area (WHPA) around a pumping well from measured breakthrough curves at said pumping well. 
- Predictor and target are generated through forward modeling from a set of hydrogeological model with different hydraulic conductivity fields (not shown).
- The predictor is the set of breakthrough curves coming from 6 different injection wells around the pumping well (Figure 3).
- The target is the WHPA (Figure 4).
  
For this example, the data is already pre-processed. We are working with 400 examples of both `d` and `h` and consider one extra pair to be predicted. See details in the reference.
  
<p align="center">
<img src="/docs/img/data/curves.png" background-color: white>
</p>
<p align="center">
  Figure 3: Predictor set. Prior in the background and test data in thick lines.
<p align="center">

  <p align="center">
<img src="/docs/img/whpa.png" background-color: white>
</p>
<p align="center">
  Figure 4: Target set. Prior in the background (blue) and test data to predict in red.
<p align="center">
  
#### Building the BEL model
In this package, a BEL model consists of a succession of Pipelines (imported from scikit-learn).
  
```python
import os
from os.path import join as jp

import joblib
import pandas as pd
from loguru import logger
from sklearn.cross_decomposition import CCA
from sklearn.decomposition import PCA
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler, PowerTransformer

import demo_visualization as myvis

from skbel import utils
from skbel.learning.bel import BEL

  ```
We can then define a function that returns our desired BEL model :

```python
def init_bel():
    """
    Set all BEL pipelines. This is the blueprint of the framework.
    """
    # Pipeline before CCA
    X_pre_processing = Pipeline(
        [
            ("scaler", StandardScaler(with_mean=False)),
            ("pca", PCA()),
        ]
    )
    Y_pre_processing = Pipeline(
        [
            ("scaler", StandardScaler(with_mean=False)),
            ("pca", PCA()),
        ]
    )

    # Canonical Correlation Analysis
    cca = CCA()

    # Pipeline after CCA
    X_post_processing = Pipeline(
        [("normalizer", PowerTransformer(method="yeo-johnson", standardize=True))]
    )
    Y_post_processing = Pipeline(
        [("normalizer", PowerTransformer(method="yeo-johnson", standardize=True))]
    )

    # Initiate BEL object
    bel_model = BEL(
        X_pre_processing=X_pre_processing,
        X_post_processing=X_post_processing,
        Y_pre_processing=Y_pre_processing,
        Y_post_processing=Y_post_processing,
        cca=cca,
    )

    return bel_model

  ```
  
- The ```X_pre_processing``` and ```Y_pre_processing``` objects are pipelines which will first scale the data for predictor and target, then apply the dimension reduction through PCA.

- The ```X_post_processing``` and ```Y_post_processing``` objects are pipelines which will normalize predictor and target CV's.
  
- Finally, the BEL model is constructed by passing as arguments all these pipelines in the `BEL` object.
  
#### Training the BEL model
A simple function can be defined to train our model.
  ```python
def bel_training(
    bel_,
    *,
    X_train_: pd.DataFrame,
    x_test_: pd.DataFrame,
    y_train_: pd.DataFrame,
    y_test_: pd.DataFrame = None,
    directory: str = None,
):
    """
    :param bel_: BEL model
    :param X_train_: Predictor set for training
    :param x_test_: Predictor "test"
    :param y_train_: Target set for training
    :param y_test_: "True" target (optional)
    :param directory: Path to the directory in which to unload the results
    :return:
    """
    #%% Directory in which to load forecasts
    if directory is None:
        sub_dir = os.getcwd()
    else:
        sub_dir = directory

    # Folders
    obj_dir = jp(sub_dir, "obj")  # Location to save the BEL model
    fig_data_dir = jp(sub_dir, "data")  # Location to save the raw data figures
    fig_pca_dir = jp(sub_dir, "pca")  # Location to save the PCA figures
    fig_cca_dir = jp(sub_dir, "cca")  # Location to save the CCA figures
    fig_pred_dir = jp(sub_dir, "uq")  # Location to save the prediction figures

    # Creates directories
    [
        utils.dirmaker(f, erase=True)
        for f in [
            obj_dir,
            fig_data_dir,
            fig_pca_dir,
            fig_cca_dir,
            fig_pred_dir,
        ]
    ]

    # %% Fit BEL model
    bel_.Y_obs = y_test_
    bel_.fit(X=X_train_, Y=y_train_)

    # %% Sample for the observation
    # Extract n random sample (target CV's).
    # The posterior distribution is computed within the method below.
    bel_.predict(x_test_)

    # Save the fitted BEL model
    joblib.dump(bel_, jp(obj_dir, "bel.pkl"))
    msg = f"model trained and saved in {obj_dir}"
    logger.info(msg)
  ```

#### Load the dataset and run everything
- The example dataset is saved as pandas DataFrame in `skbel/examples/dataset`.
- An arbitrary choice has to be made on the number of PC to keep for the predictor and the target. In this case, they are set to 50 and 30, respectively.
- The CCA operator `cca` is set to keep the maximum number of CV possible (30).
- Note that the variable `y_test` is the unknown to predict. It is simply saved within the BEL model for later uses (such as plotting or experimental design), but it is ignored during the training.
  
```python
if __name__ == "__main__":

    # Set directories
    data_dir = jp(os.getcwd(), "dataset")
    output_dir = jp(os.getcwd(), "results")

    # Load dataset
    X_train = pd.read_pickle(jp(data_dir, "X_train.pkl"))
    X_test = pd.read_pickle(jp(data_dir, "X_test.pkl"))
    y_train = pd.read_pickle(jp(data_dir, "y_train.pkl"))
    y_test = pd.read_pickle(jp(data_dir, "y_test.pkl"))

    # Initiate BEL model
    model = init_bel()

    # Set model parameters
    model.mode = "mvn"  # How to compute the posterior conditional distribution
    # Set PC cut
    model.X_n_pc = 50
    model.Y_n_pc = 30
    # Save original dimensions of both predictor and target
    model.X_shape = (6, 200)  # Six curves with 200 time steps each
    model.Y_shape = (1, 100, 87)  # One matrix with 100 rows and 87 columns
    # Number of CCA components is chosen as the min number of PC
    n_cca = min(model.X_n_pc, model.Y_n_pc)
    model.cca.n_components = n_cca
    # Number of samples to be extracted from the posterior distribution
    model.n_posts = 400

    # Train model
    bel_training(
        bel_=model,
        X_train_=X_train,
        x_test_=X_test,
        y_train_=y_train,
        y_test_=y_test,
        directory=output_dir,
    )

    # Plot raw data
    myvis.plot_results(model, base_dir=output_dir)

    # Plot PCA
    myvis.pca_vision(
        model,
        base_dir=output_dir,
    )

    # Plot CCA
    myvis.cca_vision(bel=model, base_dir=output_dir)
  ```
#### Visualization
##### PC's
  <p align="center">
<img src="/docs/img/pca_plot1-01.png" background-color: white>
</p>
<p align="center">
  Figure 5: A. The full predictor of the test set is the concatenation of all breakthrough curves (thick curves on Figure 3). PCA decomposition with 50 PC’s allows to recover the original curves while smoothing out some noise present in the original dataset. B. Original test target compared to test target projected then back-transformed with its 30 Principal Components. C. Principal Components of the predictor training set and projected test set. D. Principal Components of the target training set and the projected test set. E. Cumulative explained variance for the Principal Components of the predictor training set. F. Cumulative explained variance for the Principal Components of the target training set.
<p align="center">

##### CV's

  <p align="center">
<img src="/docs/img/cca_joint_new-01.png" background-color: white>
</p>
<p align="center">
  Figure 6: A, B, C. Canonical Variates bivariate distribution plots for the 4 firsts pairs of the training set, and the projection in the canonical space of the selected test predictor and associated test target (see notches). The posterior distribution of h^c computed according to BEL and KDE can be compared on the y marginal plot. D. Decrease of the Canonical Correlation Coefficient r with the number of CV pairs for the training set.
<p align="center">
  
##### Prediction
  <p align="center">
<img src="/docs/img/uq/818bf1676c424f76b83bd777ae588a1d_cca_30-01.png" background-color: white>
</p>
<p align="center">
  Figure 7: BEL-derived posterior predictions. The red domain corresponds to the envelope of predictions. The uncertainty reduces around the data sources (injection wells)
<p align="center">
 
### Contributing
Contributors and feedback from users are welcome. Don't hesitate to submit an issue or a PR, or request a new feature.

### References
  
[Hermans, T., Oware, E., Caers, J., 2016. Direct prediction of spatially and temporally varying physical properties from time-lapse electrical resistance data. Water Resour. Res. 52, 7262–7283.](https://doi.org/10.1002/2016WR019126)

[Scheidt, C., Li, L., Caers, J., 2018. Quantifying Uncertainty in Subsurface Systems, Geophysical Monograph Series. John Wiley & Sons, Inc., Hoboken, NJ, USA.](https://doi.org/10.1002/9781119325888)
  
[Thibaut, R., Laloy, E., Hermans, T., 2021. A new framework for experimental design using Bayesian Evidential Learning: the case of wellhead protection area. arXiv:2105.05539 [cs].](https://arxiv.org/abs/2105.05539)
