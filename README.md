# TensorFlow Determinism

The TensorFlow determinism debug tool described in the GTC 2019 talk
[_Determinism in Deep Learning_][1] will be released here. Note that the
features of this tool have not yet been released.

Updates on the status of determinism in deep learning will also be conveyed here
along with dynamic patches for TensorFlow.

## Installation

Use `pip` to install:

```
pip install tensorflow-determinism
```

This will install a package that can be imported as `tfdeterminism`. The
installation of `tensorflow-determinism` will not automatically install
TensorFlow. The intention of this is to allow you to install your chosen
version of TensorFlow. You will need to install your chosen version of
TensorFlow before you can import and use `tfdeterminism`.

## Deterministic TensorFlow Solutions

There are currently two main ways to access GPU-deterministc functionality in
TensorFlow for most deep learning applications. The first way is to use an
NVIDIA NGC TensorFlow container. The second way is to use version 1.14.0 of
stock TensorFlow with GPU support plus the application of a patch supplied in
this repo.

### NVIDIA NGC TensorFlow Containers

NGC TensorFlow container versions 19.06 (based on TensorFlow 1.13.1) and 19.07
(based on TensorFlow 1.14.0) both implement GPU-deterministic TensorFlow
functionality. In Python code running inside the container, this can be enabled
as follows:


```
import tensorflow as tf
import os
os.environ['TF_DETERMINISTIC_OPS'] = '1'
# Now build your graph and train it
```

For information about pulling and running the NVIDIA NGC containers, see [these
instructions][2].

### Stock TensorFlow

Version 1.14.0 of stock TensorFlow implements a reduced form of GPU
determinism, which must be supplemented with a patch provided in this repo.
The following Python code is running on a machine in which `pip` package
`tensorflow-gpu=1.14.0` has been installed correctly and on which
`tensorflow-determinism` has also been installed (as shown in the
[installation](#installation) section above).

```
import tensorflow as tf
from tfdeterminism import patch
patch()
# build your graph and train it
```

Stock TensorFlow with GPU support can be installed as follows:

```
pip install tensorflow-gpu=1.14.0
```

The TensorFlow project includes [detailed instructions][3] for installing
TensorFlow with GPU support.

### Additional Ingredients in the Determinism Recipe

You'll also need to set any and all appropriate random seeds:

```
os.environ['PYTHONHASHSEED']=str(SEED)
random.seed(SEED)
np.random.seed(SEED)
tf.set_random_seed(SEED)
```

If you're using Horovod for multi-GPU training, you may need to disable Tensor
Fusion (assuming that the non-determinism associated with Tensor Fusion has not
yet been resolved):

```
os.environ['HOROVOD_FUSION_THRESHOLD']='0'
```

## Detailed Status of Determinism in TensorFlow and Beyond

Confirmed and likely sources of non-determinism, along with any existing
solutions, are being tracked here.

### GPU-Specific Sources of Non-Determinism

#### Historic GPU-Specific Sources of Non-Determinism

In the past, `tf.math.reduce_sum` and `tf.math.reduce_mean` operated
non-deterministically when running on a GPU. This was resolved before
TensorFlow version 1.12. These ops now function deterministically
by default when running on a GPU.

#### Confirmed Current GPU-Specific Sources of Non-Determinism (With Solutions)

Source                                         | NGC 19.06+ Solutions | Stock TF 1.14.0+ Solutions |
:----------------------------------------------|:---------------------|:---------------------------|
TF auto-tuning of cuDNN convolution algorithms | TCD or TDO           | TCD or TDP                 |
cuDNN convolution backprop to weight gradients | TCD or TDO           | TCD or TDP                 |
cuDNN convolution backprop to data gradients   | TCD or TDO           | TCD or TDP                 |
cuDNN max-pooling backprop                     | TCD or TDO           | TCD or TDP                 |
`tf.nn.bias_add` backprop                      | TDO                  | TDP                        |
`tf.image.resize_bilinear` fwd and bwd         | NS                   | NS                         |

Key to the solutions refenced above:

Solution | Description                                                                                                                                                                                     |
:--------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
TCD      | Set environment variable `TF_CUDNN_DETERMINISTIC` to '1' or 'true'. Also *do not* set environment variable `TF_USE_CUDNN_AUTOTUNE` at all (and particularly *do not* set it to '0' or 'false'). |
TDO      | Set environment variable `TF_DETERMINISTIC_OPS` to '1' or 'true'. Also *do not* set environment variable `TF_USE_CUDNN_AUTOTUNE` at all (and particularly *do not* set it to '0' or 'false').   |
TDP      | Apply `tfdeterminism.patch`. Note that we are currently working on getting solution TDO into stock TensorFlow (see [PR 31465](https://github.com/tensorflow/tensorflow/pull/31465)).            |
NS       | There is currently no solution available for this, but one is under development.                                                                                                                |

#### Other Possible GPU-Specific Sources of Non-Determinism

Going beyond the above-mentioned sources, in version 1.12 of TensorFlow (and
also in the master branch on 2019-03-03, afer release 1.31.1), the following
files call CUDA `atomicAdd` either directly or indirectly. This makes them
candidates for the injection of non-determinism.

* `crop_and_resize_op_gpu.cu.cc`
* `scatter_functor_gpu.cu.h`
* `scatter_nd_op_gpu.cu.cc`
* `sparse_tensor_dense_matmul_op_gpu.cu.cc`
* `resize_nearest_neighbor_op_gpu.cu.cc`
* `segment_reduction_ops.h`
* `segment_reduction_ops_gpu.cu.cc`
* `dilation_ops_gpu.cu.cc`
* `maxpooling_op_gpu.cu.cc`
* `svd_op_gpu.cu.cc`
* `cuda_kernel_helper_test.cu.cc`
* `depthwise_conv_op_gpu.h`
* `resampler_ops_gpu.cu.cc`
* `histogram_op_gpu.cu.cc`
* `stateful_random_ops_gpu.cu.cc`

Unless you are using TensorFlow ops that depend on these files (i.e. ops with
similar names), then your model will not be affected by these potential sources
of non-determinism.

Beyond `atomicAdd` There are ten other CUDA [atomic functions][4] whose use
could lead to the injection of non-determinism, such as `atomicCAS` (the most
generic, atomic compare and swap). Note also that the word 'atomic' was present
in 167 files in the TensorFlow repo and some of these may be related to the use
of CUDA atomic operations. It's important to remember that it's possible to use
CUDA atomic operations without injecting non-determinism, and that, therefore,
when CUDA atomic operations are present in op code, it doesn't guarantee that
the op injects non-determinism into the computation.

### Sources of Non-Determinism in TensorFlow Unrelated to GPU

* [Issue 29101](https://github.com/tensorflow/tensorflow/issues/29101): Random
  seed not set in graph context of `Dataset#map`. This may have been resolved
  in version 1.14 of TensorFlow.
* `tf.data.Dataset` with more than one worker. The work-around is to use only
  one worker.

### Sources of Non-Determinism Beyond TensorFlow

* TensorRT timing-based kernel schedule. Each time an inference engine is
  generated, it could be slightly different, particularly if there is varying
  load on the machine used to run TensorRT. There is a solution planned for
  this.
* Horovod Tensor Fusion. Work-around: disable Tensor Fusion by setting the
  environment variable `HOROVOD_FUSION_THRESHOLD` to '0'. This issue may have
  been resolved by Horovod
  [pull-request 1130](https://github.com/horovod/horovod/pull/1130) (not yet
  confirmed).

[1]: http://bit.ly/determinism-in-deep-learning
[2]: https://ngc.nvidia.com/catalog/containers/nvidia:tensorflow
[3]: https://www.tensorflow.org/install/gpu
[4]: https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#atomic-functions