# ABCBoost

This toolkit consists of ABCBoost, a concrete implementation of [Fast ABCBoost](https://arxiv.org/pdf/2205.10927.pdf) (Fast Adaptive Base Class Boost). 

## Quick Start
### Installation guide
Run the following commands to build ABCBoost from source:
```
git clone git@github.com:pltrees/abcboost.git
cd abcboost
mkdir build
cd build
cmake ..
make
cd ..
```
This will create two executables (`abcboost_train` and `abcboost_predict`) in the `abcboost` directory.
`abcboost_train` is the executable to train models.
`abcboost_predict` is the executable to validate and inference using trained models.

The default setting builds ABCBoost as a single-thread program.  To build ABCBoost with multi-thread support [OpenMP](https://en.wikipedia.org/wiki/OpenMP) (OpenMP comes with the system GCC toolchains on Linux), turn on the multi-thread option:
```
cmake -DOMP=ON ..
make clean
make
```
Note that the default g++ in Mac may not support OpenMP.   


If we set `-DNATIVE=ON`, the compiler may better optimize the code according to specific native CPU instructions: 
```
cmake -DOMP=ON -DNATIVE=ON .. 
make clean
make
```
Again, we do not recommend to turn on this option in Mac. 


To build ABCBoost with GPU support, install [NVIDIA CUDA Toolkits](https://developer.nvidia.com/cuda-downloads) and set the option `CUDA=ON`:
```
cmake -DOMP=ON -DNATIVE=ON -DCUDA=ON ..
make clean 
make
```


### Datasets 

Four datasets are provided under `data/' folder: 

[comp_cpu](http://www.cs.toronto.edu/~delve/data/comp-activ/desc.html) for regression, in both CSV  and LIBSVM formats: `comp_cpu.train.libsvm`, `comp_cpu.train.csv`, `comp_cpu.test.libsvm`, `comp_cpu.test.csv`. Note that other tree platforms may not support the CSV format. 

[ijcnn1](https://www.csie.ntu.edu.tw/~cjlin/libsvmtools/datasets/binary.html#ijcnn1) for binary classification. 

[covtype](https://archive.ics.uci.edu/ml/datasets/covertype) for multi-class classification. Note that ABCBoost package does not require class labels to start from `0` while other platforms may require so. 

[mslr10k](https://www.microsoft.com/en-us/research/project/mslr/) for ranking. Only a small subset is provided here. 


### Lp Regression 

L2 regression (p=2) is the default, although in some datasets using p>2 achieves lower errors.  To train Lp regression on the provided `comp_cpu` dataset, with p=2:  
```
./abcboost_train -method regression -lp 2 -data data/comp_cpu.train.csv -J 20 -v 0.1 -iter 1000 
```
This will generate a model named `comp_cpu.train.libsvm_regression_J10_v0.1_p2.model` on the current directory. For testing, execute 
```
./abcboost_predict -data data/comp_cpu.test.csv -model comp_cpu.train.csv_regression_J20_v0.1_p2.model 
```
which outputs two files: (1)  `comp_cpu.test.csv_regression_J20_v0.1_p2.testlog` which stores the history of test L2 loss and Lp loss (unless p=2) for all the iterations; and (2) `comp_cpu.test.csv_regression_J20_v0.1_p2.prediction` which stores the predictions for all testing data points.  



### Binary Classification (Robust LogitBoost) 

We support both `Robust LogitBoost` and `MART`. Because `Robust LogitBoost` uses second-order information to compute the gain for tree plits, it often improves `MART`. 
```
./abcboost_train -method robustlogit -data data/ijcnn1.train.csv -J 20 -v 0.1 -iter 1000 
./abcboost_predict -data data/ijcnn1.test.csv -model ijcnn1.train.csv_robustlogit_J20_v0.1.model 
```
Users can replace `robustlogit` by `mart` to test different algorithms. 


### Multi-Class Classification (Aaptive Base Class Robust LogitBoost) 

We support four training methods: `robustlogit`,  `abcrobustlogit`, `mart`, and `abcmart`. The following example is for `abcrobustlogit` on `covtype` dataset which has `7` classes. In order to identify the `base class`, we need to specify the `-search` parameter (between 1 and 7 for this dataset) and `-gap` parameter (`5` by default) : 
```
./abcboost_train -method abcrobustlogit -data data/covtype.train.csv -J 20 -v 0.1 -iter 1000 -search 2 -gap 10
./abcboost_predict -data data/covtype.test.csv -model covtype.train.csv_abcrobustlogit2g10_J20_v0.1_w0.model 
```
In this example, the `_w0` in the model name means we set `-warmup_iter` to be 0 by default. Note that if the `-search` parameter is set to be `0`, then the `exhaustive` strategy is adopted (which is the same as `-search 7` in this example). We choose this design convention so that readers do not have to know the number of classes of the dataset. 


In practice, the test data would not have labels. The following example outputs only the predicted class labels: 
```
./abcboost_predict -data data/covtype.nolabel.test.csv -no_label 1 -model covtype.train.csv_abcrobustlogit2g10_J20_v0.1_w0.model 
```
in `covtype.nolabel.test.csv_abcrobustlogit2g10_J20_v0.1_w0.prediction` file. In many scenarios, practitioners are often more interested in the predicted class probabilities. The next example

```
./abcboost_predict -data data/covtype.nolabel.test.csv -no_label 1 -save_prob 1 -model covtype.train.csv_abcrobustlogit2g10_J20_v0.1_w0.model 
```
outputs an additional file `covtype.nolabel.test.csv_abcrobustlogit2g10_J20_v0.1_w0.probability`. 


### Ranking (LambdaRank) 

Ranking tasks are supported by using `-method lambdarank`. Note that the query/group file need to be specified (the query file tells us how many instances in the data for each query):
```
./abcboost_train -method lambdarank -data data/mslr10k.train -query data/mslr10k.train.query -J 20 -v 0.1 -iter 100 
./abcboost_predict -data data/mslr10k.test -query data/mslr10k.test.query -model mslr10k.train_lambdarank_J20_v0.1.model
```

### Feature Binning (Histograms) (`-data_max_n_bins`)


Before the training stage, each feature is preprocessed to be integers between `0` and `MaxBin-1` where we set the up limit by `-data_max_n_bins MaxBin`. Smaller `MaxBin` results in faster training, but it may hurt the accuracy if `MaxBin` is set to be too small. The default value of `-data_max_n_bins` is 1000. The following example changes this parameter to 500:
```
./abcboost_train -method robustlogit -data data/ijcnn1.train.csv -J 20 -v 0.1 -iter 1000  -data_max_n_bins 500`

```

### GPU 

If the executables are compiled with GPU support, we can specify the GPU device from the command line:
```
CUDA_VISIBLE_DEVICES=0 ./abcboost_train -method robustlogit -data data/ijcnn1.train.csv -J 20 -v 0.1 -iter 1000
```
Here we specify `GPU 0` as the device. (Use `nvidia-smi` to find out available GPUs)


### Parameters

Here we illustrate some common parameters and provide some examples:
* `-iter` number of iterations (default 1000)
* `-J` number of leaves in a tree (default 20)
* `-v` learning rate (default 0.1)
* `-search` searching size for the base class (default 1: we greedily choose the base classes according to the training loss). For example, 2 means we try the class with the greatest loss and the class with the second greatest loss as base class and pick the one with lower loss as the base class for the current iteration.
* `-n_threads` number of threads (default 1) <strong>It can only be used when multi-thread is enabled. (Compile the code with `-DOMP=ON` in cmake.)</strong>
* `-additional_files` using other files to do bin quantization besides the training data. File names are separated by `,` without additional spaces, e.g., `-additional_files file1,file2,file3`.
* `-additional_files_no_label` using other unlabeled files to do bin quantization besides the training data. File names are separated by `,` without additional spaces, e.g., `-additional_files_no_label file1,file2,file3`.

To train the model with 2000 iterations, 16 leaves per tree and 0.08 learning rate:
```
./abcboost_train -data data/covtype.train.csv -J 16 -v 0.08 -iter 2000
```

To train the model with 2000 iterations, 16 leaves per tree, 0.08 learning rate and enable the exhaustive base class searching:
```
./abcboost_train -data data/covtype.train.csv -J 16 -v 0.08 -iter 2000 -search 0 
```
Note that the exhaustive searching produces better-generalized model while requiring substantially more time. For the `covtype` dataset (which has 7 classes), using `-search 0` is effectively equivalent to `-search 7`. 

The labels in the specified additional files are not used in the training. Only the feature values are used to generate (potentially) better quantization. Better testing results may be obtained when using additional files



## More Configuration Options:
#### Data related:
* `-data_use_mean_as_missing`
* `-data_min_bin_size` minimum size of the bin
* `-data_sparsity_threshold`
* `-data_max_n_bins` max number of bins (default 1000)
* `-data_path, -data` path to train/test data
#### Tree related:
* `-tree_clip_value` gradient clip (default 50)
* `-tree_damping_factor`, regularization on numerator (default 1e-100)
* `-tree_max_n_leaves`, -J (default 20)
* `-tree_min_node_size` (default 10)
#### Model related:
* `-model_use_logit`, whether use logitboost
* `-model_data_sample_rate` (default 1.0)
* `-model_feature_sample_rate` (default 1.0)
* `-model_shrinkage`, `-shrinkage`, `-v`, the learning rate (default 0.1)
* `-model_n_iterations`, `-iter` (default 1000)
* `-model_save_every`, `-save` (default 100)
* `-model_eval_every`, `-eval` (default 1)
* `-model_name`, `-method` regression/lambdarank/mart/abcmart/robustlogit/abcrobustlogit (default abcrobustlogit)
* `-model_pretrained_path`, `-model`
#### Adaptive Base Class (ABC) related:
* `-model_base_candidate_size`, `base_candidates_size`, `-search` (default 2) base class searching size in abcmart/abcrobustlogit
* `-model_gap`, `-gap` (default 5) the gap between two base class searchings. For example, `-model_gap 2` means we will do the base class searching in iteration 1, 4, 6, ...
* `-model_warmup_iter`, `-warmup_iter` (default 0) the number of iterations that use normal boosting before ABC method kicks in. It might be helpful for datasets with a large number of classes when we only have a limited base class searching parameter (`-model_base_candidate_size`) 
* `-model_warmup_use_logit`, `-warmup_use_logit` 0/1 (default 1) whether use logitboost in warmup iterations.
* `-model_abc_sample_rate`, `-abc_sample_rate` (default 1.0) the sample rate used for the base class searching
* `-model_abc_sample_min_data` `-abc_sample_min_data` (default 2000) the minimum sampled data for base class selection. This parameter only takes into effect when `-abc_sample_rate` is less than `1.0`
#### Regression related:
* `-regression_lp_loss`, `-lp` (default 2.0) whether use Lp norm instead of L2 norm. p (p >= 1.0) has to be specified
* `-regression_use_hessian` 0/1 (default 1) whether use second-order derivatives in the regression. This parameter only takes into effect when `-regression_lp_loss p` is set and `p` is greater than `2`.
* `-regression_huber_loss`, `-huber` 0/1 (default 0) whether use huber loss
* `-regression_huber_delta`, `-huber_delta` the delta parameter for huber loss. This parameter only takes into effect when `-regression_huber_loss 1` is set
#### Parallelism:
* `-n_threads`, `-threads` (default 1)
* `-use_gpu` 0/1 (default 1 if compiled with CUDA) whether use GPU to train models. This parameter only takes into effect when the executable is complied with CUDA (i.e., the flag `-DCUDA=on` is enabled in `cmake`).
#### Other:
* `-save_log`, 0/1 (default 0) whether save the runtime log to file
* `-save_model`, 0/1 (default 1)
* `-no_label`, 0/1 (default 0) It should only be enabled to output prediction file when the testing data has no label in test
* `-test_auc`, 0/1 (default 0) whether compute AUC in test
* `-stop_tolerance` (default 2e-14) It works for all non-regression tasks, e.g., classification. The training will stop when the total training loss is less than the stop tolerance.
* `-regression_stop_factor` (default 1e-5) The auto stopping criterion is different from the classification task because the scale of the regression target is unknown. We adaptively set the regression stop tolerate to `regression_stop_factor * total_loss / sum(y^p)`, where `y` is the regression targets and `p` is the value specified in `-regression_lp_loss`.
* `-regression_auto_clip_value` 0/1 (default 1) whether use our adaptive clipping value computation for the predict value on terminal nodes. When enabled, the adaptive clipping value is computed as `tree_clip_value * max_y - min_y` where `tree_clip_value` is set via `-tree_clip_value`, `max_y` and `min_y` are the maximum and minimum regression target value, respectively.
* `-gbrank_tau` (default 0.1) The tau parameter for gbrank.

## R Support
We provide an R library to enable calling ABCBoost subroutines from R.
To build and install the library, type the following command in `abcboost/`:
```
cd ..
R CMD build abcboost
```

For users' convience, we also provide, under `R/`,  the pre-built `abcboost_1.0.0.tar.gz` and `abcboost_1.0.0_mult.tar.gz`, for single-thread version and multi-thread version, respectively. To install the (single-thread) package, in R console, type 
```
install.packages('R/abcboost_1.0.0.tar.gz', repos = NULL, type = 'source')
```
One can use `setwd` to change the current working directory. Note that we should first remove the package (`remove.packages('abcboost')`) if we hope to replace the single-thread version with the multi-thread version. 


Function description (no need to copy to console):
```
# No need to copy to console
# abcboost_train: (train_Y,train_X,model_name,iter,leaves,shinkage,search=2,gap=5,params=NULL)
# abcboost_test: (test_Y,test_X,model,params=NULL)
# abcboost_predict: (test_X,model,params=NULL)
# abcboost_save_model: function(model,path)
# abcboost_load_model: function(path)
```
Here we show an example of training and testing:
```
library(abcboost)
data <- read.csv(file='data/covtype.train.csv',header=FALSE)
X <- data[,-1]
Y <- data[,1]
data <- read.csv(file='data/covtype.test.csv',header=FALSE)
testX <- data[,-1]
testY <- data[,1]
# The last argument of abcboost_train is optional. 
# We use n_threads as an example
# All command line supported parameters can be passed via list: 
# list(parameter1=value1, parameter2=value2,...)
model <- abcboost_train(Y,X,"abcrobustlogit",100,20,0.1,2,5,list(n_threads=1))
# abcboost_save_model(model,'mymodel.model')
# model <- abcboost_load_model('mymodel.model')
res <- abcboost_test(testY,testX,model)
# predict without label 
res <- abcboost_predict(testX,model)
# We also provide a method to read libsvm format data into sparse array
data <- abcboost_read_libsvm('data/covtype.train.libsvm')
X <- data$X
Y <- data$Y
data <- abcboost_read_libsvm('data/covtype.test.libsvm')
testX <- data$X
testY <- data$Y
# X can be a either a dense matrix or a sparse matrix
# The interface is the same as the dense case, 
# but with better performance for sparse data
model <- abcboost_train(Y,X,"abcrobustlogit",100,20,0.1,2,5,list(n_threads=1))
res <- abcboost_test(testY,testX,model)
```


## Matlab Support
We provide a Matlab wrapper to call ABCBoost subroutines from Matlab.
To compile the Matlab mex files in Matlab:
```
cd src/matlab
compile    % single-thread version 
```
or 
```
cd src/matlab
compile_mult_thread 
```

One can use `mex -setup cpp` in a matlab console to check the specified C++ compiler. 


For the convenience of users, we provide the compiled executables for both Linux and Windows under `matlab/linux` and `matlab/windows` respectively. 



Assume the executables are stored in `matlab/linux` or `matlab/windows`. Here we show an example of training and testing:
```
tr = load('../../data/covtype.train.csv');
te = load('../../data/covtype.test.csv');
Y = tr(:,1);
X = tr(:,2:end);
testY = te(:,1);
testX = te(:,2:end);

params = struct;
params.n_threads = 1;
% The params argument is optional. 
% We use n_threads as an example
% All command line supported parameters can be passed via params: params.parameter_name = value
model = abcboost_train(Y,X,'abcrobustlogit',100,20,0.1,1,0,params);
% abcboost_save(model,'mymodel.model');
% model = abcboost_load('mymodel.model');
params.test_auc = 1;
res = abcboost_test(testY,testX,model,params);
% predict without label 
res = abcboost_predict(testX,model,params);

% Sparse matlab matrix is also supported
% For example, we included the libsvmread.c from the LIBSVM package for data loading
[Y, X] = libsvmread('../../data/covtype.train.libsvm');
[testY, testX] = libsvmread('../../data/covtype.test.libsvm');
% Here X and testX are sparse matrices
model = abcboost_train(Y,X,'abcrobustlogit',100,20,0.1,1,0,params);
res = abcboost_test(testY,testX,model);
```

## Python Support
We provide the python support through `pybind11`.
Before the compilation, `pybind11` should be installed:

`python3 -m pip install pybind11`

To compile the single-thread version on Linux:
```
cd python/linux
bash compile_py.sh
```
After the compilation, a shared library `abcboost.so` is generated.

Make sure `abcboost.so` is in the current directory.
Paste the following code in a `python3` interactive shell:
```
import numpy as np
# We use a matrix-format sample data here
data = np.genfromtxt('../../data/covtype.train.csv',delimiter=',').astype(float)
#
Y = data[:,0]
X = data[:,1:]
data = np.genfromtxt('../../data/covtype.test.csv',delimiter=',').astype(float)
testY = data[:,0]
testX = data[:,1:]
import abcboost
model = abcboost.train(Y,X,'abcrobustlogit',100,20,0.1,3,0)
# All command line supported parameters can be passed as optional keyword arguments
# For example:
# model = abcboost.train(Y,X,'abcrobustlogit',100,20,0.1,3,0,n_threads=1)
# abcboost.save(model,'mymodel.model')
# model = abcboost.load('mymodel.model')
res = abcboost.test(testY,testX,model)
% predict without label 
res = abcboost.predict(testX,model)
# Alternatively, we also support libsvm-format sparse matrix
# We use sklearn to load libsvm format data as a scipy.sparse matrix
# sklearn can be installed as: python3 -m pip install scikit-learn
import sklearn
import sklearn.datasets
# X is a scipy.sparse matrix
[X, Y] = sklearn.datasets.load_svmlight_file('../../data/covtype.train.libsvm')
[testX, testY] = sklearn.datasets.load_svmlight_file('../../data/covtype.train.libsvm')
# The training and testing interfaces are unified for both dense and sparse matrices
model = abcboost.train(Y,X,'abcrobustlogit',100,20,0.1,3,0)
res = abcboost.test(testY,testX,model)
```


## References
* Li, Ping. "ABC-Boost: Adaptive Base Class Boost for Multi-Class Classification." _ICML_ 2009.
* Li, Ping. "Robust LogitBoost and Adaptive Base Class (ABC) LogitBoost." _UAI_ 2010.
* Li, Ping and Zhao, Weijie. "Fast ABC-Boost: A Unified Framework for Selecting the Base Class in Multi-Class Classification." _arXiv preprint arXiv:2205.10927_ 2022.

## Copyright and License
ABCBoost is provided under the Apache-2.0 license.
