# FP8 Tutorial
> **NOTE** The 8-bit floating-point (FP8) data types are only featured on Hopper GPUs or newer (>= SM90) and require CUDA 11.8 or higher.

This document will demonstrate two approaches on how to use the FP8 types in JAX and TensorFlow using the native XLA, without any library dependencies: The high-level API for a quick turnaround, and the low-level API for most flexibility and control.

# How to Use the High-Level FP8 API

This repo provides a collection of customized layers in TF and JAX
to help utilize the 8-bit floating-point precision (FP8) data types on Hopper GPUs for
better performance and lower memory utilization in both training and inference applications.

There are multiple ways to take advantage of FP8, such as the custom
FP8 kernel-based methods (e.g., [Transformer
Engine](https://github.com/NVIDIA/TransformerEngine)) or the native-XLA compiler-based
method, which is the subject of this repo.

To help the use of XLA-FP8, in this repo, we provide two high-level APIs for using FP8, namely the `fp8_module.Dense` and `fp8_module.DenseGeneral` layers which are a drop-in replacement for
`keras.layers.Dense` (TensorFlow) and `flax.linen.DenseGeneral` (JAX)
respectively. You can directly place them into your XLA JIT-compiled functions to
carry out computation in FP8.

## A TensorFlow Example

Using XLA-FP8 in TF is simple and users only need to replace the
`tf.keras.layers.Dense` with `fp8_module.Dense` in their JIT-compiled model as shown
in the example below.

```python
@tf.function(jit_compile=True)
def train_step(x):
  #y = tf.keras.layers.Dense(16)(x)
  y = fp8_module.Dense(16)(x)
```

If you do not JIT your functions, you will also need to set an extra environment variable to invoke autojit.
```bash
TF_XLA_FLAGS="--tf_xla_auto_jit=2" 
```

# How to Use the Low-Level FP8 API
Using the low-level FP8 API allows for maximum flexibility and control over how and where to use the FP8 operations. Currently we have only integrated FP8 support in matrix-multiplication operations when using NVIDIA Hopper FP8 in JAX and TensorFlow through XLA.

## FP8 Datatypes
JAX and TensorFlow both support two flavors of FP8, E4M3, and E5M2. E4M3 (with 4 bits for the exponent and 3 bits for mantissa) has twice the resolution of E5M2 but half the range. Because the gradient of model variables tend to fluctuate more so than the variables, gradients can use the extra range offered by E5M2. Therefore E4M3 is the recommended datatype for the forward pass and E5M2 for the backward pass. More details on this further down.

<table>
<tr>
<td>TensorFlow</td><td>JAX</td>
</tr>

<tr>
<td>

```python
import tensorflow as tf
from tensorflow.python.framework import dtypes
# dtypes.float8_e4m3fn
# dtypes.float8_e5m2
```
</td>
<td>

```python
import jax.np as jnp
# jnp.float8_e4m3fn
# jnp.float8_e5m2
```
</td>
</tr>
</table>

## Basic FP8 Concepts
The FP8 datatypes in JAX and TensorFlow can only be created by down-casting from a wider type. The example below demonstrates how to do this:

```python
import tensorflow as tf
from tensorflow.python.framework import dtypes
f8e4m3 = dtypes.float8_e4m3fn

a = tf.constant([1.2345678, 2.3456789, 3.4567891], dtype=dtypes.float16)
print(a)
# tf.Tensor([1.234 2.346 3.457], shape=(3,), dtype=float16)

# You can generate FP8 data by downcasting.
a_fp8 = tf.cast(a, f8e4m3)
print(a_fp8)
# tf.Tensor([1.25 2.25 3.5], shape=(3,), dtype=float8_e4m3fn)
```

Due to the limited range of the FP8 datatypes, you will need to scale the higher-precision data to keep it below the upper bound of what FP8 can represent (`E4M3_max = 448` and `E5M2_max = 57344`). The quantization recipe for converting a higher-precision data to FP8 is:

- Take the maximum of the absolute value of the wide-type data, `amax = reduce_max(abs(data))`.
- Calculate a scaling factor so that it maps `amax` close to the upper bound of FP8 (e.g. `E4M3_max` or `0.9 * E4M3_max` to buffer for data fluctuation, say across the training steps), `scale = amax / E4M3_max`.
- Divide the wider-type data by the scaling factor, `scaled_data = data / scale`.
- Saturate the value by clamping to the FP8-representable range, `saturated_date = clamp(scaled_data, -E4M3_max, E4M3_max)`.
- Finally downcast the safely-scaled data to FP8, `fp8_data = cast(saturated_date, F8E4M3)`.

The de-quantization recipe is the inverse of the above:

- Upcast the FP8 data to a higher-precision, `unscaled_data = cast(fp8_data, FP16 or FP32)`
- Restore the scale, `data = unscaled_data * scale`.
- Recalculate the scaling factor, `scale = reduce_max(abs(data)) / E4M3_max`.

The quantization/de-quantization operations or QDQ for short is the most fundamental concept that you will need to equip arbitrary models with FP8.

## Use FP8 in JIT-compiled GEMM Operations

Let's put all of the above ideas into practice and carry out a matrix-multiplication in FP8:

```python
import tensorflow as tf
from tensorflow.python.framework import dtypes
f8e4m3 = dtypes.float8_e4m3fn
E4M3_max = 448

# Create or load the wide-type data.
a = tf.random.uniform((16, 64), dtype=dtypes.float16)
b = tf.random.uniform((64, 16), dtype=dtypes.float16)

# Calculate the scaling factors, which are always stored in wider types.
a_scale = tf.reduce_max(tf.abs(a)) / E4M3_max
b_scale = tf.reduce_max(tf.abs(b)) / E4M3_max

# Convert to FP8.
a_fp8 = tf.cast(a / a_scale, f8e4m3)
b_fp8 = tf.cast(b / b_scale, f8e4m3)

# JIT your model, which takes-in both the FP8 data along with scaling factors.
@tf.function(jit_compile=True)
def matmul_fp8(a_fp8, a_scale, b_fp8, b_scale):
    # Up-cast from FP8 to a wider type.
    a = tf.cast(a_fp8, dtypes.float16) * a_scale
    b = tf.cast(b_fp8, dtypes.float16) * b_scale
    c = tf.matmul(a, b) # Result in FP16.
    return c

c = matmul_fp8(a_fp8, a_scale, b_fp8, b_scale)
print(c)
```

The `matmul_fp8` function takes two tensors in FP8 and two scaling factors in FP32 as scalars. It then seemingly first upcasts the tensors to the FP16 types and multiplies by the scaling factors before calling the `tf.matmul` operation. You may be wondering, "Doesn't the upcasting defeat the purpose of using FP8?" Yes it would have, if the XLA compiler were to follow the instructions as is, but it does not! Instead, the XLA compiler recognizes this pattern of upcasting and scaling (and many of its variants) and instead elides the extraneous operations and seamlessly calls the FP8 variant of the fused `matmul` operation, passing in the original FP8 data along with the scaling factors.

## Delayed Scaling and More Complex Patterns
Calculating the scaling factor is expensive as it needs computing the `amax` by taking the `reduce_max` of the absolute value of the wide-type data. A proven emperical solution is to not compute the `amax` prior to using the data in a GEMM computation (as done in the example above), rather fuse the `amax` computation wih the GEMM operation, thereby eliminating the `amax` latency. But how can we proceed with the operations such as `tf.cast(a_fp8, dtypes.float16) * a_scale` without apriori knowledge of `a_scale`? Assuming the data fluctuation from one training step to the next is not wildly different, we can use the `amax` as computed by the previous training step in the current training step. Employing this delayed-scaling recipe would simplify the above code as follows:

```python
# Create or load the wide-type data.
a = tf.random.uniform((16, 64), dtype=dtypes.float16)
b = tf.random.uniform((64, 16), dtype=dtypes.float16)

# Begin with factors of unity. The factors need to have the same type as the wide-type data.
# The first few training steps will warm up and adjust the scaling factors.
a_scale = tf.constant(1.0, dtypes.float16)
b_scale = tf.constant(1.0, dtypes.float16)
c_scale = tf.constant(1.0, dtypes.float16)

# Convert to FP8.
a_fp8 = tf.cast(a, f8e4m3)
b_fp8 = tf.cast(b, f8e4m3)

# JIT your model, which takes-in both the FP8 data along with scaling factors.
# Note that we now also pass the (delayed) scaling factor for the output.
@tf.function(jit_compile=True)
def matmul_fp8(a_fp8, a_scale, b_fp8, b_scale, c_scale):
    # Up-cast from FP8 to a wider type.
    a = tf.cast(a_fp8, dtypes.float16) * a_scale
    b = tf.cast(b_fp8, dtypes.float16) * b_scale

    # Call the GEMM operation.
    c = tf.matmul(a, b)

    # Quantize the output. These steps need to be followed exactly.
    # We clip before casting to ensure proper saturation and protect against overflow.
    saturated_c = tf.clip_by_value(c / c_scale, -E4M3_max, E4M3_max)
    c_fp8 = tf.cast(saturated_c, f8e4m3)
    new_c_scale = tf.reduce_max(tf.abs(c)) / E4M3_max

    # Return the new scaling factors along with the results.
    # The new scaling factors will be used in the next training step.
    return c_fp8, new_c_scale

for step in training_steps:
  a_fp8, a_scale, b_fp8, b_scale = other_computation(...)
  c_fp8, c_scale = matmul_fp8(a_fp8, a_scale, b_fp8, b_scale, c_scale)

print(c_fp8)
print(c_scale)
```

In the above example, the XLA compiler will recognize the entirety of the pattern and 
- Elide the upcasting and scaling prior to the matmul like in the previous example.
- Elide the clipping, descaling, downcasting and `amax` computation.
- And instead fuse the GEMM operation with that of computing `amax` for maximal performance.

## Amax Rolling History
The result of a binary FP8 operation is a wide-type (FP16 or FP32), which needs to be scaled and converted back to FP8 before it leaves the streaming microprocessor (SM) and written back to memory, or else we risk losing the memory-IO advantages of the FP8 datatype. This means that we will need to use an apriori value for the scaling factor as explained above. However, exactly because of the estimated nature of the scaling factor, some of the FP8 results may inevitably fall outside the FP8 range and be clipped off. In order to reduce the chances of this clipping error to occur, we can keep track of, not just the `amax` value at the previous step, but a moving window of previous `amax` values. That way we can use the `max(amax_history)` as the effective `amax` for scaling purposes. When a new `amax` is computed, we will roll out the oldest `amax` in the history and insert the new value at the top of the queue in a first-in first-out (FIFO) ring buffer fashion.

For the details of how to implement full-fledged FP8 JAX layers in ~ 100 lines of code, see [DenseFp8.py](JAX/DenseFp8.py) or [dense.py](tensorflow/fp8_module/dense.py) for the case of TensorFlow. The included Python library has incorporated all of the above concepts and can be used as a guide for developing even more complex FP8 pipelines in addition to being used as a functioning library.

## FP8 Fusions
We have also added support for various FP8 fusions, such as GEMM + Bias (vector and matrix) + Relu. This will allow for obviating memory writebacks and further increases performance.
