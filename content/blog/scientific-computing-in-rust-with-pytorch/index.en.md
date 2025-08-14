+++
title = "Scientific Computing in Rust"
description = "Just use PyTorch in Rust for now"
draft = false

weight = 10

[taxonomies]
tags = ["Rust"]

[extra]
feature_image = "rusty-torch.png"
feature = true
+++

There's a joke that says, if you want a good recommendation on the Internet, instead of asking "what's good for X", just say "nothing is good for X. Prove me wrong". Following that joke, I'd say "no Rust crates are really suitable and capable for scientific computing" :) And as a Rust fan, I really wish someone could prove me wrong.

Enough joking, my problem is: I've been doing tensor networks for quantum simulations recently. While Python is one of my favorite languages (the other is Rust), I really want a good type system to help me a bit when my code gets complicated, so naturally I want to turn to Rust for a little help. However, I could not find a crate, a pure Rust one that supports complex numbers, fast fourier transforms _and_ matrix decompositions (like LU, SVD or QR decompositions).

My solution is [`tch-rs`](https://github.com/LaurentMazare/tch-rs), a Rust binding of PyTorch. Below is some of my experience and learning of it.

> Before that, a quick **announcement**:
>
> I made a project template for anyone who want to try out scientific computing in Rust without hassles. In my repo [scientific-computing-rs](https://github.com/ifsheldon/scientific-computing-rs), I use [`uv`](https://github.com/astral-sh/uv) to manage PyTorch and other Python dependencies with some proper configurations, so you can just clone my repo and everything should just work!
>

## Survey on Scientific Computing Crates

Before using `tch-rs`, I've done some research about scientific computing crates (including some for AI), but none of Rust native ones are as feature complete. Here is a summary：

|     Crates/Features      | Linalg - Matrix Decompositions | Backprop - Builtin Optimizers, Autodiff | Complex Number | Inter-Tensor Parallelism\* |
| :----------------------: | :----------------------------: | :-------------------------------------: | :------------: | :------------------------: |
|         nalgebra         |               ✅                |                    ❌                    |       ✅        |             ✅              |
|           burn           |               ❌                |                    ✅                    |       ❌        |            ✅^1             |
|          candle          |               ❌                |                    ✅                    |       ❌        |             ✅              |
| tch-rs (non-Rust-native) |               ✅                |                    ✅                    |       ✅        |            ❌^2             |

> \* Inter-Tensor Parallelism: For this, I mean the capability to allow parallel processing of different _independent_ tensors with multiple threads. More on this later.

> Inter-Tensor Parallelism Benchmark (on my 16-core Mac):
>
> 1. Parallel computation of `burn` tensors has ~4x speed-up 
> 2. Parallel computation of `tch` tensors has ~1.3x speed-up 
>
> while the reference speed-up (with `nalgebra` or `candle`) is ~11x.
>

## Stumble into `tch-rs`

Unlike PyTorch in Python, `tch-rs` lacks comprehensive documentations, tutorials or examples. One of the biggest missing piece in the documentation is how to simply create tensors that require gradients, which is surprising given that this is how most of us step into PyTorch.

Compared to PyTorch, `tch-rs` has an additional concept, `VarStore`, which "stores variables used by one or multiple layers" with a device. Another addition is `Path`, whose documentation is "a variable store with an associated path for variables naming", which is a bit confusing in the beginning. Nevertheless, for a simple compute graph like $y = x^2$ with no deep neural networks, to calculate $\frac{\partial x}{\partial y}$, we should do something like this:

```rust
use tch::nn::{Adam, VarStore};
use tch::{Device, Kind, Tensor};

let vs = VarStore::new(Device::Cpu);
let root = vs.root();
// with an optimizer
let mut adam = Adam::default().build(&vs, 1e-1).unwrap();
// create a variable under a path, here `root`
let v = root.randn("v", &[2, 2, 2, 2], 0.0, 1.0);
let v_copy = v.copy();
v_copy.print();
let loss = v.pow_tensor_scalar(2).sum(Kind::Float);
loss.backward();
// or without an optimizer, however update tensors based on gradients you like
let gradient = v.grad();
assert!(gradient.allclose(&(2 * v.copy()), 1e-5, 1e-5, false));
adam.step();
adam.zero_grad();
v.print();
assert!(!v.allclose(&v_copy, 1e-5, 1e-5, false));
```

Compared to a few lines of the PyTorch equivalent code (which is too trivial to be written down here), we need more verbosity, but that's the price we need to pay in exchange for some help from a powerful type checker. Having said that, I believe the APIs can be cleaner. According to `tch-rs`, these APIs are generated from [ocaml-torch](https://github.com/LaurentMazare/ocaml-torch), so maybe we should just spend a bit more time and effort to design APIs, making them feel more Rusty.

### More on Parallelism

When we talk about parallelism in PyTorch, we often refer to parallel computing on accelerators (like GPUs and NPUs). This is the major workhorse in deep learning (and more generally machine learning), of course. Such parallelism in PyTorch is acheived by specialized operators and the actual computation is often delegated to C++ code and kernels in batched manner. However, there's more than such operator-based parallelism. Specifically, we can use threads (either on CPU or on GPU). With the global interpreter lock (GIL), there's no true multithreading in Python. Although there is ongoing effort to remove GIL, the ecosystem of Python has yet to adapt to that change. This means we cannot do something trivial like the below in Python for now:

```
tensors = [....] # tensors of different shapes, cannot be stacked and batch-processed

for each thread i:
    take tensor_i in tensors
    perform some tensor operations on tensor_i
```

Luckily, we don't have GIL in Rust. With fearless concurrency, in theory, we can do such inter-tensor parallelism with ease - just spawning multiple threads and taking care of any issues about `Sync` and/or `Send`. However, as shown in the above table, `tch-rs` does not support inter-tensor parallelism while other crates support it more or less. 

Here's my code for testing the speedup from inter-tensor parallelism of `tch-rs`:

```rust
use rayon::prelude::*;
use std::hint::black_box;
use std::time::Instant;
use tch::{Device, Kind, Tensor, set_num_interop_threads, set_num_threads};

fn main() {
    let tensor_size = 200;
    let num_tensors = 2000;
    let repeat = 20;
    benchmark_parallelism_tch(tensor_size, num_tensors, repeat);
}

fn get_randn_tensors_tch(tensor_size: usize, num_tensors: usize) -> Vec<Tensor> {
    let tensor_size = tensor_size as i64;
    (0..num_tensors)
        .map(|_| Tensor::randn([tensor_size, tensor_size], (Kind::Float, Device::Cpu)))
        .collect()
}

fn benchmark_parallelism_tch(tensor_size: usize, num_tensors: usize, repeat: usize) {
    let core_num = num_cpus::get();
    set_num_threads(core_num as i32);
    set_num_interop_threads(core_num as i32);
    println!("core_num: {core_num}");
    rayon::ThreadPoolBuilder::new()
        .num_threads(core_num)
        .build_global()
        .unwrap();
    // warmup
    let _results: f64 = get_randn_tensors_tch(tensor_size, core_num)
        .into_par_iter()
        .map(|t| t.sum(Kind::Float).double_value(&[]))
        .sum();
    // prevent optimizing out the above line
    black_box(_results);

    let avg_time_parallel = (0..repeat)
        .map(|_| {
            let tensors = get_randn_tensors_tch(tensor_size, num_tensors);
            let now = Instant::now();
            let _sum: f64 = tensors
                .into_par_iter()
                .map(|t| t.sum(Kind::Float).double_value(&[]))
                .sum();
            let duration = now.elapsed();
            black_box(_sum);
            duration.as_secs_f64()
        })
        .sum::<f64>()
        / repeat as f64;

    let avg_time_serial = (0..repeat)
        .map(|_| {
            let tensors = get_randn_tensors_tch(tensor_size, num_tensors);
            let now = Instant::now();
            let _sum: f64 = tensors
                .into_iter()
                .map(|t| t.sum(Kind::Float).double_value(&[]))
                .sum();
            let duration = now.elapsed();
            black_box(_sum);
            duration.as_secs_f64()
        })
        .sum::<f64>()
        / repeat as f64;

    println!(
        "avg_time_parallel: {avg_time_parallel}s, avg_time_serial: {avg_time_serial}s, speedup: {}x",
        avg_time_serial / avg_time_parallel
    );
}
```

> For complete code, see my repo [scientific-computing-rs](https://github.com/ifsheldon/scientific-computing-rs).

My suspect is that `VarStore` is to be blamed. Indeed, if we dig into the code

```rust
#[derive(Debug)]
pub struct VarStore {
    pub variables_: Arc<Mutex<Variables>>,
    device: Device,
    kind: Kind,
}
```

there's a `Arc<Mutex<Variables>>` which may serve as a "GIL" for tensors stored in a `VarStore`. I didn't have time to dig into that further, but I think it's worth investigating. 

As the documentation of `tch-rs` mention that "the code generation part for the C api on top of libtorch comes from ocaml-torch", probably this lock is necessary only in the OCaml bindings, but we don't know whether PyTorch's APIs overfit Python with GIL too much to be safe to call without locks. Would be nice if an expert in PyTorch internals can share some insights in [my issue in tch-rs repo](https://github.com/LaurentMazare/tch-rs/issues/966).

## Some Thoughts

I've seen several frameworks for scientific computing and/or ML in Rust, including those that are not listed in my table, so I'd say the ecosystem is diverse. However, it seems most of them are rebuilding wheels of different shapes while the APIs of them are not crafted, or functions are lacking, or maintenance is suspended. I think we should take more from the success of PyTorch. For scientific computing and ML, instead of rebuilding wheels with engineering innovations, probably we should focus on user experience first. It's super hard to maintain high-quality math and ML code, not to mention complicated GPU-accelerated kernels. For scientific computing crates, probably we can just reused as many routines and kernels from PyTorch as possible and focus on crafting APIs and core algorithms. We don't need to throw away existing wheels to build a faster and more comfortable car.


## Metadata

Version: 0.0.1

Date: 2025.08.07

License: [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)