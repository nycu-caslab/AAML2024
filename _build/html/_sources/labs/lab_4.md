# Lab 4 : Elementwise Unit

## Goal of this lab
---
- [Porting and Profiling the Models - 10%](#porting-and-profiling-the-models-10)
- [Accelerating the Logistic Function - 60%](#accelerating-the-logistic-function-60)
- [Accelerating the Softmax Function - 20%](#accelerating-the-softmax-function-20)
- [Questions in the Demo - 10%](#questions-in-the-demo-10)

## Introduction
---
Modern models frequently utilize specialized activation functions that involve complex mathematical computations, such as exponentials, square roots, and reciprocals. Unlike simpler functions like ReLU, these sophisticated operations can become bottlenecks during model inference. In this lab, we will design an element-wise unit specifically for enhancing the inference speed of the **Logistic** and **Softmax** functions in the **MobileViT** model.

## Porting and Profiling the Models - 10%
---
### Porting the New Model and Tests - 5%

In this lab, we provide you a new model and the project template. Follow the instrusctions below to install them.

1. Project Template with Tests

    The project template contains some tests and files you'll use in this lab.
    ```sh
    $ cd ${CFU_ROOT}/proj/
    $ wget https://github.com/nycu-caslab/AAML2024/raw/main/lab4_util/lab4_template.zip
    $ unzip lab4_template.zip
    ```

    In the `/src/tensorflow/lite/micro/` directory, you’ll find tests for the Logistic and Softmax functions, along with some updated kernels for MobileViT inference. **DO NOT MODIFY** any files in this directory.

    ```{versionchanged} 10/18,23:25
    Updated the template to include modified kernels.
    ```

2. MobileViT Model

    This is the model we aim to accelerate. We will use this model to benchmark the performance of your design.
    ```sh
    $ cd ${CFU_ROOT}/common/src/models
    $ wget https://github.com/nycu-caslab/AAML2024/raw/main/lab4_util/lab4_model.zip
    $ unzip lab4_model.zip
    ```

After that, just like we did in lab 1, we should modify some files to add the new models: 

`CFU-Playground/common/src/models/models.c`  
```c
#include "models/ds_cnn_stream_fe/ds_cnn.h"
// add codes below
#include "models/mobileViT_xxs/mobileViT.h"

...

#if defined(INCLUDE_MODEL_DS_CNN_STREAM_FE)
        MENU_ITEM(AUTO_INC_CHAR, "Ds cnn stream fe", ds_cnn_stream_fe_menu),
#endif
// add codes below
#if defined(INCLUDE_MODEL_MOBILE_VIT_XXS)
        MENU_ITEM(AUTO_INC_CHAR, "MobileViT xxs", mobileViT_xxs_menu),
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
```

`CFU-Playground/proj/<lab4 proj folder>/Makefile`  
(This has been done in the project template.)
```sh
DEFINES += INCLUDE_MODEL_MOBILE_VIT_XXS
```

After completing these steps, you should be able to run the MobileViT model and the Tests in the `Project menu`.
```sh
$ make prog EXTRA_LITEX_ARGS="--cpu-variant=perf+cfu"
$ make load
```
* MobileViT result of zeros input without any modification
```
 Counter |  Total | Starts | Average |     Raw
---------+--------+--------+---------+--------------
    0    |     0  |     0  |   n/a   |            0
    1    |     0  |     0  |   n/a   |            0
    2    |     0  |     0  |   n/a   |            0
    3    |     0  |     0  |   n/a   |            0
    4    |     0  |     0  |   n/a   |            0
    5    |   145M |     9  |    16M  |    145244066
    6    |  1318M |    37  |    36M  |   1317968736
    7    |     0  |     0  |   n/a   |            0
  5234M (   5233667340 )  cycles total
0 : -128,
1 : 67,
2 : 57,
3 : -72,
4 : 100,
5 : 37,
6 : -9,
7 : -8,
8 : -6,
9 : -21,
```
* Tests for Logistic and Softmax
```
CFU Playground
==============
 1: TfLM Models menu
 2: Functional CFU Tests
>3: Project menu
 4: Performance Counter Tests
 5: TFLite Unit Tests
 6: Benchmarks
 7: Util Tests
 8: Embench IoT
 d: Donut demo
main> 3

Running Project menu

Project Menu
============
 1: Run logistic tests
 2: Run softmax tests
 x: eXit to previous menu
project> 1

Running Run logistic tests

LOGISTIC TEST:
Testing LogisticQuantizedInt8AroundZeroShouldMatchGolden
Testing LogisticQuantizedInt8NarrowRangeShouldMatchGolden
Testing LogisticQuantizedInt8BasicShouldMatchGolden
Testing LogisticQuantizedInt8WideRangeShouldMatchGolden
4/4 tests passed
~~~ALL TESTS PASSED~~~
```

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
`CFU-Playground/third_party/tflite-micro/tensorflow/lite/kernels/internal/reference/integer_ops/logistic.h`  
Then, you will find the mathematical computations are defined in:  
`CFU-Playground/third_party/tflite-micro/third_party/gemmlowp/fixedpoint/fixedpoint.h`
```

## Accelerating the Logistic Function - 60%
---

\begin{gather*}
\text{logistic}(x) = \frac{1}{1 + e^{-x}}
\end{gather*}

(We have done these for you in the template.)  
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
  perf_enable_counter(6);

  ...

  perf_disable_counter(6);
}
```
````

### Evaluation Criteria
You should first pass the `LOGISTIC TEST` in the `Project menu`, then run MobileViT with any input and obtain scores based on the criteria.

| Cycles of Counter 6 | > 1000M | 400~1000M | 140~400M | < 140M |
| ------------------- | ------- | --------- |:-------- |:------ |
| Score               | 0       | 10        | 30       | 60     |

```{hint}
You may need to accelerate both the exponential and reciprocal parts in order to meet the full score cycle count criteria.
```

```{attention} 
You will get **0%** if you can't pass all `LOGISTIC TEST` in the `Project menu`.
```

### Guide

```{note}
**You are not required to follow to the provided guide below**. Instead, feel free to use any method to accelerate the inference of Logistic function, provided that it passes the unit test.
```
First of all, it is essential to familiarize yourself with **fixed-point** arithmetic and the `FixedPoint` class defined in `fixedpoint.h`. The key components and functions you are likely to use within the `FixedPoint` class include `kIntegerBits`, `FromRaw()`, and `raw()`.

Additionally, You can refer to [Wikipedia - Q format notation](https://en.wikipedia.org/wiki/Q_(number_format)) and the comments for the `FixedPoint` class in `fixedpoint.h` to help you understand fixed-point arithmetic. **The [Q format converter](https://chummersone.github.io/qformat.html#converter) is also a very useful tool** that you can use to convert between fixed-point and floating-point numbers, which will definitely assist you in debugging.

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
For the hardware unit section, it’s important to first familiarize yourself with the **val/rdy interface** of the CFU. In Lab 2, the calculation required only a single cycle, making it straightforward to handle. However, in this lab, your CFU operations may take more than one cycle to complete. To better understand how to manage multi-cycle operations, refer to this article for examples and guidance.
> [Details and Use Cases of the CPU <-> CFU interface](https://cfu-playground.readthedocs.io/en/latest/interface.html)

For calculating results using hardware, you have the option to employ either a Lookup Table or Mathematical Approximation methods, such as the Taylor Series, Newton-Raphson division, or Polynomial Approximation.

In this lab, for the **exponential function, we recommend using either a Lookup Table or the Taylor Series**. For the **reciprocal function, we suggest using a Lookup Table or the Newton-Raphson division method**, similar to the approach used in `one_over_one_plus_x_for_x_in_0_1()`, but implemented in hardware.

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

You should first pass the `SOFTMAX TEST` in the `Project menu`, then run MobileViT with any input and obtain scores based on the criteria.

| Cycles of Counter 5 | > 60M  | 30~60M  | < 30M |
| ------------------- | ------ |:------- |:----- |
| Score               | 0      | 10      | 20    |

```{attention} 
You will get **0%** if you can't pass all `SOFTMAX TEST` in the `Project menu`.
```

### Guide

Following the previous approach, replacing the exponential and reciprocal functions with CFU operations is a good idea. The CFU operations designed for the Logistic function might work here too. 

But, be careful: the fixed-point integer bits used for the exponential function are different in the Logistic and Softmax functions—**the Logistic uses 4 and the Softmax uses 5**. Pay close attention to this detail.

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