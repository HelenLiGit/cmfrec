# Collective Matrix Factorization

Implementation of collective matrix factorization, based on _Relational learning via collective matrix factorization_, with some enhancements and alternative models for cold-start recommendations as described in _Cold-start recommendations in Collective Matrix Factorization_.

This is a hybrid collaborative filtering model for recommender systems that takes as input either explicit item ratings or implicit-feedback data, and side information about users and/or items (although it can also fit pure collaborative-filtering and pure content-based models). The overall idea was extended here to also be able to do cold-start recommendations (for users and items that were not in the training data but which have side information available).

Although the package was developed with recommender systems in mind, it can also be used in other domains - just take any mention of users as rows in the main matrix and any mention of items as columns.

For more information about the implementation here, or if you would like to cite this in your research, see ["Cold-start recommendations in Collective Matrix Factorization"](https://arxiv.org/abs/1809.00366)

For a similar package with Poisson distributions see [ctpfrec](https://github.com/david-cortes/ctpfrec).

Written in C with a Python interface. R version to come in the future.

## Update 2020-03-20

The package has been rewritten in C with Python wrappers. If you've used earlier versions of this package which relied on Tensorflow for the calculations (and before that, Casadi), the optimal hyperparameters will be very different now as it has changed some details of the loss function such as not dividing some terms by the number of entries.

The new version is faster, multi-threaded, and has some new functionality, but if for some reason you still need the old one, it can be found under the git branch "tensorflow".

## Highlights

* Can fit factorization models with or without user and/or item side information.
* Can fit the usual explicit-feedback model as well as the implicit-feedback model with weighted binary entries (see [3]).
* Can be used for cold-start recommendations (when using side information).
* Supports user and item biases in the explicit-feedback models.
* Provides an API for top-N recommended lists.
* Can use either an alternating least-squares procedure (ALS) or a gradient-based procedure using an L-BFGS optimizer (depending on the specific model) (the package bundles a modified version of [Okazaki's C implementation](https://github.com/chokkan/liblbfgs)).
* Provides a content-based model and other models aimed at better cold-start recommendations.
* Provides an intercepts-only "most-popular" model for non-personalized recommendations, which can be used as a benchmark as it takes similar hyperparameters as the other models.
* Allows variations of the original collective factorization models such as setting some factors to be used only for one factorization, setting different weights for the errors on each matrix, or setting different regularization parameters for each matrix.
* Can use sigmoid transformations for binary-distributed columns in the side info data.
* Can work with both sparse and dense matrices for each input (e.g. can also be used as a general missing-value imputer for 2-d data).
* Supports observation weights (for the explicit-feedback models).

## Basic idea

The model consist in predicting the rating (weighted confidence for implicit-feedback case) that a user would give to an item by performing a low-rank factorization of an interactions matrix (e.g. ratings) `X ~ A * t(B)`, using side information about the items (such as movie tags) and/or users (such as their demographic info) by also factorizing the item side info matrix and/or the user side info matrix `U ~ A * t(C), I ~ B * t(D)`, sharing the same item/user-factor matrix used to factorize the ratings, or sharing only some of the latent factors.

This also has the side effect of allowing recommendations for users and items for which there is side information but no ratings, although these predictions might not be as high quality.

Alternatively, can produce factorizations in wich the factor matrices are determined from the attributes directly (e.g. `A = U * C`), with or without a free offset.

## Instalation

Linux, MacOS + GCC, Windows depending on `setuptools`:
```
pip install cmfrec
```
or
```
pip install git+https://www.github.com/david-cortes/cmfrec.git
```

Windows with unlucky `setuptools`: clone or download this repository and then install with `setup.py` - e.g.:
```
git clone https://www.github.com/david-cortes/cmfrec.git
cd cmfrec
python setup.py install
```
(Requires package `findblas`, which can be install with `pip install findblas`)


As it contains C code, it requires a C compiler. On Windows, this usually means it requires a Visual Studio Build Tools installation (with MSVC140 component for conda, or MinGW + GCC), and if using Anaconda, might also require configuring it to use said Visual Studio instead of MinGW.

On Mac, installing this package will first require getting OpenMP modules for the default clang compiler (redistributions from apple don't come with this essential component, even though clang itself does fully support it), or installing gcc (by default, apple systems will alias gcc to clang, which causes problems).

**Note:** this package relies heavily on BLAS and LAPACK functions for calculations. It's recommended to use MKL (comes by default in Anaconda) or OpenBLAS as backend for them, but note that, as of OpenBLAS 0.3.9, some of the functions used here might be significantly faster in MKL depending on CPU architecture.

## Sample Usage

```python
import numpy as np, pandas as pd
from cmfrec import CMF_explicit

### Generate random data
n_users = 4
n_items = 5
n_ratings = 10
n_user_attr = 4
n_item_attr = 5
k = 3
np.random.seed(1)

### 'X' matrix (note that you can also pass it as SciPy COO)
ratings = pd.DataFrame({
    "UserId" : np.random.randint(n_users, size=n_ratings),
    "ItemId" : np.random.randint(n_items, size=n_ratings),
    "Rating" : np.random.normal(loc=3., size=n_ratings)
})
### 'U' matrix
user_info = pd.DataFrame(np.random.normal(size = (n_users, n_user_attr)))
user_info["UserId"] = np.arange(n_users)
### 'I' matrix
item_info = pd.DataFrame(np.random.normal(size = (n_items, n_item_attr)))
item_info["ItemId"] = np.arange(n_items)

### Fit the model
model = CMF_explicit(method="als", k=k)
model.fit(X=ratings, U=user_info, I=item_info)

### Predict rating that user 3 would give to items 2 and 4
model.predict(user=[3, 3], item=[2, 4])

### Top-5 highest predicted for user 3
model.topN(user=3, n=5)

### Top-5 highest predicted for user 3, if it were a new user
model.topN_warm(X_col=ratings.ItemId.loc[ratings.UserId == 3],
                X_val=ratings.Rating.loc[ratings.UserId == 3],
                n=5)

### Top-5 highest predicted for user 3, based on side information
model.topN_cold(U=user_info.iloc[[3]], n=5)
```

Users and items can be reindexed internally (if passing data frames), so you can use strings or non-consecutive numbers as IDs when passing data to the object's methods.

For more details see the online documentation.

## Getting started

For a longer example with real data see the notebook [MovieLens Recommender with Side Information](http://nbviewer.jupyter.org/github/david-cortes/cmfrec/blob/master/example/cmfrec_movielens_sideinfo.ipynb).

## Documentation

Documentation is available at ReadTheDocs: [http://cmfrec.readthedocs.io/en/latest/](http://cmfrec.readthedocs.io/en/latest/).

## Some comments

This kind of model requires a lot more hyperparameter tuning that regular low-rank matrix factorization, and fitting a model with badly-tuned parameters might result in worse recommendations compared to discarding the side information.

If your dataset is larger than the MovieLens ratings, adding product side information is unlikely to add more predictive power, but good user side information might still be valuable.

## Troubleshooting

For any installation problems or errors encountered with this software, please open an issue in this GitHub page with a reproducible example, the error message that you see, and description of your setup (e.g. Python version, NumPy version, operating system).

## References

* [1] Cortes, David. "Cold-start recommendations in Collective Matrix Factorization." arXiv preprint arXiv:1809.00366 (2018).
* [2] Singh, Ajit P., and Geoffrey J. Gordon. "Relational learning via collective matrix factorization." Proceedings of the 14th ACM SIGKDD international conference on Knowledge discovery and data mining. ACM, 2008.
* [3] Hu, Yifan, Yehuda Koren, and Chris Volinsky. "Collaborative filtering for implicit feedback datasets." 2008 Eighth IEEE International Conference on Data Mining. Ieee, 2008.
