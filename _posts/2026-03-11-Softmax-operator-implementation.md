---
title: Softmax算子实现
date: 2026-03-11 23:44:27 +0800
categories: [AI Infra, 推理框架]
tags: [CUDA, 算子]
pin: false
math: false
mermaid: false
---

## 简介

实现了Softmax算子

<!-- more -->

## 正文
基于开源项目[KuiperLLama](https://github.com/zjhellofss/KuiperLLama)，实现了Softmax算子。
主体分为两部分：
* `softmax_kernel_inplace_cu`，接口函数。框架在`kDeviceCUDA`类型的设备上进行softmax计算时，调用这个函数
* `row_softmax_f32`，核函数。进行具体的softmax算子的实现，其基本执行流程如下：
	1. 以大小为4将block分成若干pack
	2. 遍历寻找`max(x)`，然后同步
	3. 遍历计算`e^(x_i-max(x))`，并累加得到sum，然后同步
	4. 计算最终的softmax值，并写回
重点介绍一下 `row_softmax_f32`的实现细节：
* 函数开头的`template <int32_t BLOCK_DIM>`声明了一个模板类型`BLOCK_DIM`，因为`cub`库强制要求在**代码编译阶段**就必须知道线程数到底是多少，以便提前分配好硬件资源。所以通过模板类型声明了一个占位符，编译器看到`row_softmax_f32<128>(...)`这个调用时，会用128替换占位符
* 待续...
具体代码如下：
```c++
template <int32_t BLOCK_DIM>  
static __global__ void row_softmax_f32(float* input, int size) {  
  const int tid = threadIdx.x;  
  
  constexpr int pack_size = 4;  
  const int pack_num = size / pack_size;  
  const int pack_off = pack_size * pack_num;  
  
  // 遍历找最大x_i  
  float max_x = std::numeric_limits<float>::min();  
  float4* in_pack = reinterpret_cast<float4*>(input);  
  for (int i = tid; i < pack_num; i += blockDim.x) {  
    float4 in_float4 = *(in_pack + i);  
    float max_1 = fmaxf(in_float4.x, in_float4.y);  
    float max_2 = fmaxf(in_float4.z, in_float4.w);  
    max_x = fmaxf(max_x, max_1);  
    max_x = fmaxf(max_x, max_2);  
  }  
  // 尾巴  
  for (int i = pack_off + tid; i < size; i += blockDim.x) {  
    max_x = fmaxf(max_x, input[i]);  
  }  
  // 归约max_x  
  using BlockReduce = cub::BlockReduce<float, BLOCK_DIM>;  
  __shared__ typename BlockReduce::TempStorage temp;  
  __shared__ float shared_val;  
  max_x = BlockReduce(temp).Reduce(max_x, cub::Max());  
  if (threadIdx.x == 0) {  
    shared_val = max_x;  
  }  
  __syncthreads();  
  max_x = shared_val;  
  
  // 遍历，计算e^(x_i-max(x))  
  float sum = 0.0f;  
  for (int i = tid; i < pack_num; i += blockDim.x) {  
    float4 in_float4 = *(in_pack + i);  
    in_float4.x = expf(in_float4.x - max_x);  
    in_float4.y = expf(in_float4.y - max_x);  
    in_float4.z = expf(in_float4.z - max_x);  
    in_float4.w = expf(in_float4.w - max_x);  
    sum += in_float4.x;  
    sum += in_float4.y;  
    sum += in_float4.z;  
    sum += in_float4.w;  
    *(in_pack + i) = in_float4;  
  }  
  // 尾巴  
  for (int i = pack_off + tid; i < size; i += blockDim.x) {  
    float temp = expf(input[i] - max_x);  
    sum += temp;  
    input[i] = temp;  
  }  
  
  sum = BlockReduce(temp).Sum(sum);  
  if (threadIdx.x == 0) {  
    shared_val = sum;  
  }  
  __syncthreads();  
  sum = shared_val;  
  
  // 计算最终的softmax值  
  for (int i = tid; i < pack_num; i += blockDim.x) {  
    float4 in_float4 = *(in_pack + i);  
    *(in_pack + i) = make_float4(in_float4.x / sum, in_float4.y / sum, in_float4.z / sum, in_float4.w / sum);  
  }  
  
  for (int i = pack_off + tid; i < size; i += blockDim.x) {  
    float inv_sum = 1.0f / sum;  
    input[i] = input[i] * inv_sum;  
  }  
}  
void softmax_kernel_inplace_cu(const tensor::Tensor& input, void* stream) {  
  CHECK(!input.is_empty());  
  
  CHECK(input.device_type() == base::DeviceType::kDeviceCUDA);  
  
  const int32_t size = static_cast<int32_t>(input.size());  
  float* in_ptr = const_cast<float*>(input.ptr<float>());  
  constexpr int threads_num = 128;  
  if (stream) {  
    cudaStream_t stream_ = static_cast<cudaStream_t>(stream);  
    row_softmax_f32<128><<<1, threads_num, 0, stream_>>>(in_ptr, size);  
  }  
  else {  
    row_softmax_f32<128><<<1, threads_num>>>(in_ptr, size);  
  }  
}

```
