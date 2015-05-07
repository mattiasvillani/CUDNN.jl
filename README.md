# CUDNN

<!---
[![Build Status](https://travis-ci.org/denizyuret/CUDNN.jl.svg?branch=master)](https://travis-ci.org/denizyuret/CUDNN.jl)
--->

## Overview

This is a Julia wrapper for the NVIDIA cuDNN GPU accelerated deep
learning library.  It consists of a low level interface and a high
level interface.  

The low level interface wraps each function in libcudnn.so in a Julia
function in libcudnn.jl, each data type in cudnn.h in a Julia datatype
in types.jl.  These were generated semi-automatically using
[Clang](https://github.com/ihnorton/Clang.jl) and are well documented
in the [cuDNN Library User Guide](https://developer.nvidia.com/cuDNN).

The high level interface introduces some Julia datatypes and provides
reasonable defaults for arguments when possible.  I will mostly
describe the high level interface below.  I kept the original names
from the C library and provided more convenient type signatures,
return values, and keyword arguments with reasonable defaults for the
high level interface.  All low level arguments are supported by the
high level interface using keyword arguments, however only the most
useful ones are documented below.  Please see CUDNN.jl for the
complete interface.

## Types

We introduce two data types: Tensor and Filter.  Tensors and Filters
are almost identical data structures except there is no stride option
for the filter constructor.  CUDNN docs say "Filters layout must be
contiguous in memory."  We introduce AbstractTensor as their parent
for common operations, and employ
[CudaArray](https://github.com/JuliaGPU/CUDArt.jl)'s for their data.
```
abstract AbstractTensor
immutable Tensor <: AbstractTensor; data::CudaArray; desc::cudnnTensorDescriptor_t; end
immutable Filter <: AbstractTensor; data::CudaArray; desc::cudnnFilterDescriptor_t; end
```

Tensors and Filters can be constructed using Array's, CudaArray's, or
by specifying the element type and dimensions.
```
Tensor(T::Type, dims::Dims), Filter(T::Type, dims::Dims)
Tensor(T::Type, dims::Integer...), Filter(T::Type, dims::Integer...)
Tensor(a::Array), Filter(a::Array)
Tensor(a::CudaArray), Filter(a::CudaArray)
```

Currently only Float32 and Float64 are supported for element types,
and only 4-D Tensors and Filters are supported by the majority of
CUDNN functions.  5-D Tensor operations (to support 3-D point clouds)
are under development.

The default order of Tensor dimensions in the C library documentation
is NCHW, with W being the fastest changing dimension.  These stand for
number of images, channels, height and width for image applications.
C is row-major whereas Julia is column major.  So the default size of
Tensors in CUDNN.jl are (W,H,C,N) with W being fastest changing
dimension.  Similarly the default order of Filter dimensions in Julia
are (W,H,C,K) standing for width, height, number of input feature
maps, and number of output feature maps respectively.

The following array operations are supported for Tensors and Filters:
`eltype`, `ndims`, `size`, `strides`, `stride`, `zeros`, `ones`,
`similar`, `copy`.  Also `to_host(a::AbstractTensor)` can be used to
retrieve the contents of a Tensor or Filter in a regular Julia array.


## Convolution

`cudnnConvolutionForward(src::Tensor, filter::Filter, [dest::Tensor])`
This function computes and returns dest, the convolution of src with
filter under default settings (no padding, stride=1).  For more
options please see ConvolutionDescriptor in CUDNN.jl and in the C
library documentation.  For 2-D images if src has size (W,H,C,N) and
filter has size (X,Y,C,K), the output dest will have size
(W-X+1,H-Y+1,K,N) .  If dest is not specified it will be allocated.

`cudnnConvolutionBackwardBias(src::Tensor, [dest::Tensor])` Assume
y=w*x+b where x is the forward input to a convolution layer, w is a
filter, b is the bias vector, * denotes convolution, and + denotes
broadcast addition.  Given src=dJ/dy this function computes and
returns dest=dJ/db which is simply the sum of dJ/dy across each
channel, i.e. dest=sum(src,(1,2,4)).  It is assumed that there is a
single scalar bias for each channel, i.e. the same number is added to
every pixel of every image for that channel after the convolution.
For 2-D images if src has size (W,H,C,N), dest will have size
(1,1,C,1).  If dest is not specified it will be allocated.

`cudnnConvolutionBackwardFilter(src::Tensor, diff::Tensor, grad::Filter)`

`cudnnConvolutionBackwardData(filter::Filter, diff::Tensor, grad::Tensor)`


## Pooling

`PoolingDescriptor`

`cudnnPoolingForward(pd::PoolingDescriptor, src::Tensor, dest::Tensor)`

`cudnnPoolingBackward(pd::PoolingDescriptor, src::Tensor, srcDiff::Tensor, dest::Tensor, destDiff::Tensor)`


## Activation Functions

`cudnnActivationForward(src::Tensor, [dest::Tensor])` applies a neural
network activation function (relu by default) to src and writes the
result to dest.  dest is optional and the operation is performed
in-place on src if dest is not specified.  The type of activation
function can be specified using the `mode` keyword argument.
Currently supported modes are `CUDNN_ACTIVATION_RELU`,
`CUDNN_ACTIVATION_SIGMOID`, and `CUDNN_ACTIVATION_TANH`.

`cudnnActivationBackward(src::Tensor, srcDiff::Tensor, dest::Tensor,
[destDiff::Tensor])` computes the loss gradient of the input to the
activation function from the gradient of the output of the activation
function.  If y=f(x) where f is the forward activation function and J
is loss, the arguments would be src=y, srcDiff=dJ/dy, dest=x, and
destDiff=dJ/dx.  destDiff is optional, srcDiff will be overwritten if
destDiff is not specified.  The default activation function is relu
but others can be specified using the `mode` keyword argument similar
to `cudnnActivationForward`.

`cudnnSoftmaxForward(src::Tensor, [dest::Tensor])` treats the entries
in src as unnormalized log probabilities and produces normalized
probabilities in dest.  The src and dest tensors have the same
dimensionality.  If dest is not specified, src is written in-place.
The optional keyword argument `mode` specifies over which entries the
normalization is performed.  Given a src tensor with dimensions
(W,H,C,N) mode=CUDNN_SOFTMAX_MODE_INSTANCE (default) normalizes per
image (N) across the dimensions C,H,W; i.e. computes
`dest=exp(src)./sum(exp(src),(1,2,3))` after which
`sum(dest[:,:,:,n])==1.0` for all n.  If
mode=CUDNN_SOFTMAX_MODE_CHANNEL, the normalization is performed per
spatial location (H,W) per image (N) across the dimension C,
i.e. `dest=exp(src)./sum(exp(src), 3)` after which
`sum(dest[w,h,:,n])==1.0` for all w,h,n.  In the typical use case where
size(src) is (1,1,C,N), giving unnormalized probabilities for N
instances and C classes both modes would compute the same answer.

`cudnnSoftmaxBackward(src::Tensor, srcDiff::Tensor,
[destDiff::Tensor])` Let us assume x was the input (unnormalized log
probabilities) to SoftmaxForward and y was the output (normalized log
probabilities).  SoftmaxBackward takes src=y, srcDiff=dJ/dy, and
computes destDiff=dJ/dx.  The softmax loss function is J=-log(y1)
where y1 is the probability the model assigns to the correct answer
(here assumed to be 1).  Some calculus shows dJ/dy1 = -1/y1 and dJ/dyi
= 1/y1 for i!=1.  Some more calculus shows dJ/dx1=y1-1 and dJ/dxi=yi
for i!=1.  Unfortunately cudnn gives us twice these answers and I
don't know where the factor of 2 comes from.  I recommend dividing
dJ/dyi with 2 for srcDiff to get the correct derivatives.  You should
also divide all dJ/dyi by the number of instances (N) so that the step
size is not effected by the batch size.

## Other Tensor Functions

`cudnnTransformTensor(alpha::Number, src::Tensor, beta::Number,
dest::Tensor)` computes alpha * src + beta * dest and places the
result in dest.  Both beta and dest are optional and are set to 0 and
ones(src) respectively if not specified.

`cudnnAddTensor(mode::cudnnAddMode_t, alpha::Number, bias::Tensor,
beta::Number, src::Tensor)` please refer to the C library
documentation to see what different add modes do.

`cudnnSetTensor(src::Tensor, value::Number)` sets each element of the
src Tensor to value.  `fill!(src, value)` is defined to call this
function.

`cudnnScaleTensor(src::Tensor, alpha::Number)` scales each element of
the src Tensor with alpha.  `scale!(src, alpha)` is defined to call
this function.

