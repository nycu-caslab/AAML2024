# Lab 4 : Elementwise Unit

## Goal of this lab
---
- [Porting and Profiling the Models - 10%](#porting-and-profiling-the-models-20)
- [Accelerating the Logistic Function - 60%](#accelerating-the-logistic-function-60)
- [Accelerating the Softmax Function - 20%](#accelerating-the-softmax-function-20)
- [Questions in the Demo - 10%](#questions-in-the-demo-10)

## Introduction
---
Modern models frequently utilize specialized activation functions that involve complex mathematical computations, such as exponentials, square roots, and reciprocals. Unlike simpler functions like ReLU, these sophisticated operations can become bottlenecks during model inference. In this lab, we will design an element-wise unit specifically for enhancing the inference speed of the **Logistic** and **Softmax** functions in the **MobileViT** model.

## Porting and Profiling the Models - 10%
---
### Porting the New Models - 5%

In this lab, we will provide you two models:

1. Logistic Test Model

    This model serves as a benchmark to verify the correctness of your design. In other words, your design should pass the golden test of this model in order to get the score.

2. MobileViT

    This is the actual model we aim to accelerate. We will use this model to benchmark the performance of your design.

> [model download link](https://drive.google.com/file/d/1_aUHWVC8kYe3A0T-A1ACcONjoYsGji_F/view?usp=sharing)

After downloading and unzipping the files, place the two model folders directly into the `CFU-Playground/common/src/models` directory.

Just like we did in lab 1, we should modify some files to add the new models: 

`CFU-Playground/common/src/models/models.c`  
(You can also directly use the `models.c` file provided in the .zip archive.)  
```c
#include "models/ds_cnn_stream_fe/ds_cnn.h"
// add codes below
#include "models/mobileViT_xxs/mobileViT.h"
#include "models/logistic_test/logistic_test.h"

...

#if defined(INCLUDE_MODEL_DS_CNN_STREAM_FE)
        MENU_ITEM(AUTO_INC_CHAR, "Ds cnn stream fe", ds_cnn_stream_fe_menu),
#endif
// add codes below
#if defined(INCLUDE_MODEL_MOBILE_VIT_XXS)
        MENU_ITEM(AUTO_INC_CHAR, "MobileViT xxs", mobileViT_xxs_menu),
#endif
#if defined(INCLUDE_MODEL_LOGISTIC)
        MENU_ITEM(AUTO_INC_CHAR, "Logistic test", logistic_test_model_menu),
#endif
```

`CFU-Playground/common/src/tflite.cc`
```cpp
#ifdef INCLUDE_MODEL_DS_CNN_STREAM_FE
    2048 * 1024,
#endif
// add codes below
#ifdef INCLUDE_MODEL_MOBILE_VIT_XXS
    16384 * 1024,
#endif
#ifdef INCLUDE_MODEL_LOGISTIC
    50 * 1024,
#endif
```

`CFU-Playground/proj/<lab4 proj folder>/Makefile`
```sh
DEFINES += INCLUDE_MODEL_MOBILE_VIT_XXS
DEFINES += INCLUDE_MODEL_LOGISTIC
```

Also, please follow this guide to **add the new operations required by the MobileViT**.
> [Add ops in CFU_Playground](https://hackmd.io/@Tsai-Wooo/SkjTbXdJ0)

After completing these steps, you should be able to run the two new models and obtain the scores for this section.

### Profiling the MobileViT - 5%

After executing the MobileViT model, you may notice that the inference process is quite slow. Please **analyze and compare the time consumed by each operation**, with particular attention to the **activation functions**. You may present your analysis in any format, such as **a table or a pie chart**. Additionally, you have the option to use data with or without SIMD MAC acceleration.

Here is an example of a pie chart:  
<img src="images/lab4/pie_example.png" width="600px">

```{hint}
You can use either the perf counter or the ticks displayed in the results after execution as your data source. 
the displayed ticks look something like this:

    ...
    450,LOGISTIC,1848
    451,MUL,84
    452,CONCATENATION,35
    453,CONV_2D,27022
    454,LOGISTIC,1804
    455,MUL,84
    456,CONV_2D,9782
    457,LOGISTIC,7071
    458,MUL,333
    459,AVERAGE_POOL_2D,84
    460,RESHAPE,6
    461,FULLY_CONNECTED,48
    Perf counters not enabled.
      5558M (   5558234259 )  cycles total

```

Next, trace the code of the Logistic function to identify the **complex mathematical computations involved**, which may be contributing to the slow execution.

Make sure to present your findings to the TAs during the demo.
```{hint}
You can start tracing the code from this file, and the model will use the topmost overloaded `Logistic(...)` function:  
`CFU-Playground/third_party/tflite-micro/tensorflow/lite/kernels/internal/reference/integer_ops/logistic.h` Then, you will find the mathematical computations are defined in:  
`CFU-Playground/third_party/tflite-micro/third_party/gemmlowp/fixedpoint/fixedpoint.h`
```

## Accelerating the Logistic Function - 60%
---

\begin{gather*}
\text{logistic}(x) = \frac{1}{1 + e^{-x}}
\end{gather*}

Add the integer version of the Logistic function to your project.
```sh
$ cp \
  ../../third_party/tflite-micro/tensorflow/lite/kernels/internal/reference/integer_ops/logistic.h \
  src/tensorflow/lite/kernels/internal/reference/integer_ops/logistic.h
```
````{important}
Add the **sixth perf counter** at the beginning and end of the **topmost** Logistic function within the `logistic.h` file in your project. This step is crucial for evaluating your score, so please ensure it is not overlooked.

```cpp
inline void Logistic(int32_t input_zero_point, int32_t input_range_radius,
                     int32_t input_multiplier, int32_t input_left_shift,
                     int32_t input_size, const int8_t* input_data,
                     int8_t* output_data) {
  // Integer bits must be in sync with Prepare() function.
  perf_enable_counter(6);

  ...

  perf_disable_counter(6);
}
```
````

### Evaluation Criteria

```{attention} 
You will get **0%** if you can't pass the golden test of the Logistic Test Model.
```

| Cycles of Counter 6 | > 1000M | 400~1000M | 140~400M | < 140M |
| ------------------- | ------- | --------- |:-------- |:------ |
| Score               | 0       | 20        | 40       | 60     |

```{hint}
You may need to accelerate both the exponential and reciprocal parts in order to meet the full score cycle count criteria.
```

### Guide

```{note}
**You are not required to follow to the provided guide below**. Instead, you are encouraged to use any method to accelerate model inference, provided that it passes the golden test of the Logistic Test Model.
```
First of all, it is essential to familiarize yourself with **fixed-point** arithmetic and the `FixedPoint` class defined in `fixedpoint.h`. The key components and functions you are likely to use within the `FixedPoint` class include `kIntegerBits`, `FromRaw()`, and `raw()`.  
Additionally, You can refer to [Wikipedia - Q format notation](https://en.wikipedia.org/wiki/Q_(number_format)) and the comments for the `FixedPoint` class in `fixedpoint.h` to help you understand fixed-point arithmetic. The [Q format converter](https://chummersone.github.io/qformat.html#converter) is also a very useful tool that you can use to convert between fixed-point and floating-point numbers, which will assist you in debugging.

```cpp
// Part 2: the FixedPoint class.

// A FixedPoint object represents a fixed-point value stored in the underlying
// integer type tRawType, if tRawType is a plain scalar integer type.
// Alternatively, tRawType may be a SIMD type (e.g. NEON int32x4_t) in which
// case a FixedPoint object represents a corresponding SIMD vector of fixed
// point values.
//
// tIntegerBits describes the range of the fixed-point format: if
// tIntegerBits == m then the range of representable values is the half-open
// interval [-2^m; 2^m) where the open boundary on the right side means that
// 2^m is not representable (how close the maximum representable value is to
// it, depends on bit-depth of tRawType).
//
// In "Q format notation",
//   https://en.wikipedia.org/wiki/Q_(number_format)
// we are describing the format
//   Qm.n
// where
//   m = tIntegerBits
// and
//   n = NumberOfBits(tRawType) - (m + 1)
// Note that the (m + 1) in the above line is because we adopt the convention
// that we count the integer bits exclusively of the sign bit; so (m + 1) is
// the total number of integer bits inclusive of the sign bit.
//
// Accordingly, the number of integral representable values in our range
//   [-2^m ; 2^m)
// is equal to 2^(m+1).
```

After tracing the code, you may notice that the **exponential** and the **reciprocal** are the primary bottlenecks, so we can focus on accelerating these two operations in this function.  
`third_party/gemmlowp/fixedpoint/fixedpoint.h`
```cpp
// Returns logistic(x) = 1 / (1 + exp(-x)) for x > 0.
template <typename tRawType, int tIntegerBits>
FixedPoint<tRawType, 0> logistic_on_positive_values(
    FixedPoint<tRawType, tIntegerBits> a) {
  return one_over_one_plus_x_for_x_in_0_1(exp_on_negative_values(-a));
}
```

#### Software

Below is an example of replacing the original software-computed exponential with the CFU operation:
```cpp
template <typename tRawType, int tIntegerBits>
FixedPoint<tRawType, 0> exp_on_negative_values(
    FixedPoint<tRawType, tIntegerBits> a) {
  typedef FixedPoint<tRawType, 0> ResultF;
  ...
  b = cfu_op0(0, 0, 0);
  return b;
}
```
This is just a simple example. You should properly design your CFU op to pass the data needed for the hardware unit. Also, you should properly handle the conversion of `FixedPoint` and the `tRawType` using `FromRaw()` and `raw()`.

```{note}
Given the symmetry of the Logistic function, it's only necessary to consider either positive or negative inputs for the exponential computation, with the negative ones being simpler to manage.
```

#### Hardware
As for the hardware unit section, you should first familiarize yourself with the CFU handshake signals.
> [Details and Use Cases of the CPU <-> CFU interface](https://cfu-playground.readthedocs.io/en/latest/interface.html)

Here is a dummy example:
```verilog
assign cmd_ready = ~rsp_valid & ~calculating;
always @(posedge clk) begin
    if (reset) begin
        ...
    end else if (rsp_valid) begin
        rsp_valid <= ~rsp_ready;
    end else if (cmd_valid) begin
        if (...) begin
            ...
            input_valid <= 1;
            calculating <= 1;
        end
        ...
    end
    else if (input_valid) begin
        input_valid <= 0;
    end
    else if (finish == 1) begin
        calculating <= 0;
        rsp_valid <= 1'b1;
        rsp_payload_outputs_0 <= result;
    end
end
```

For calculating results using hardware, you have the option to employ either a Lookup Table or Mathematical Approximation methods, such as the Taylor Series, Newton-Raphson division, or Polynomial Approximation.

In this lab, for the **exponential function, we recommend using either a Lookup Table or the Taylor Series**. For the **reciprocal function, we suggest using a Lookup Table or the Newton-Raphson division method**, similar to the approach used in one_over_one_plus_x_for_x_in_0_1(), but implemented in hardware.

## Accelerating the Softmax Function - 20%

\begin{gather*}
\text{softmax}(x_i) = \frac{e^{x_{i}}}{\sum_{j=1}^n e^{x_{j}}}
\end{gather*}

Add the Softmax function to your project.
```sh
$ cp \
  ../../third_party/tflite-micro/tensorflow/lite/kernels/internal/reference/softmax.h \
  src/tensorflow/lite/kernels/internal/reference/softmax.h
```

````{important}
Add the **fifth perf counter** at the beginning and end of the **second** Softmax function within the `softmax.h` file in your project. This step is crucial for evaluating your score, so please ensure it is not overlooked.

```cpp
// Quantized softmax with int8_t/uint8_t input and int8_t/uint8_t/int16_t
// output.
template <typename InputT, typename OutputT>
inline void Softmax(const SoftmaxParams& params,
                    const RuntimeShape& input_shape, const InputT* input_data,
                    const RuntimeShape& output_shape, OutputT* output_data) {
  perf_enable_counter(5);
  
  ...

  perf_disable_counter(5);
}
```
````

### Evaluation Criteria

| Cycles of Counter 5 | > 100M | 30~100M | < 30M |
| ------------------- | ------ |:------- |:----- |
| Score               | 0      | 10      | 20    |

### Guide

Following the previous approach, replacing the exponential and reciprocal functions with CFU operations is a good idea. The CFU operations designed for the Logistic function might work here too. 

But, be careful: the fixed-point integer bits used for the exponential function are different in the Logistic and Softmax functions—**the Logistic uses 4 and the Softmax uses 5**. Pay close attention to this detail.

```{note}
You'll need to use the designs from Lab 3 and Lab 4 in Lab 5, so I recommend not fully utilizing all the hardware resources in this lab. Otherwise, you might run into issues with insufficient hardware resources in Lab 5.
```

## Questions in the Demo - 10%

You will be asked several questions about the concepts covered in this lab and your implementation. This section accounts for 10% of the total lab score.


## Submission

You need to hand in your **CFU-Playground project folder** without the `build` folder and renamed with your student ID. 

Please organize your submission files into a zip archive structured as follows:
```
YourID.zip
    └── YourID/
        ├── src/
        │    ├── folder... 
        │    └── files...
        ├── cfu.v
        └── Makefile
```

```{important}
TAs should be able to run your project without any modification. If TAs cannot compile or run your code, **you won't receive any points, even if you passed the DEMO**. Also, **PLAGIARISM is not allowed**.
```