---
layout: post
title:  "Tips, tricks and gotchas in PyTorch"
date:   2019-02-06 00:53:41 +0200
categories: pytorch
author: Chang
---

*Many parts of this post are based on the [PyTorch 0.4 migration guide](https://pytorch.org/blog/pytorch-0_4_0-migration-guide/). Be sure to check it out, it's very useful.*

I've been working with PyTorch for over a year now, and I've noticed a number of little details one should care about when developing machine learning models in PyTorch. This serves as a sort of a checklist for me, but I also hope some of these points help beginners at avoiding common pitfalls.


## 1. Optimize GPU memory usage

As you may know, bigger batch sizes are more efficient to compute on the GPU. If your model or data are high in dimension, it is usually worthwhile to make sure that you have the largest possible batch size that fits in the GPU memory. To find out your programs memory usage, you can use [`torch.cuda.memory_allocated()`](https://pytorch.org/docs/stable/cuda.html#torch.cuda.memory_allocated) or look at the output of `nvidia-smi`.

Alternatively, the following hacky snippet automatically adjusts the batch size to a level where it fits in memory. Be sure to start with a slightly too large `batch_size`.

```python
training_loader = DataLoader(..., batch_size=batch_size)

for input, output in training_loader:
    try:
        opt.zero_grad()
        pred = model(input)
        loss = loss_fn(output, pred)
        loss.backward()
        opt.step()
    except RuntimeError: # CUDA: Out of memory is a generic RTE
        batch_size -= 1
        training_loader = DataLoader(..., batch_size=batch_size)
```

## 2. Avoid memory leaks via summary statistics

There's a subtle memory leak in the following code:

```python
    mean_loss = 0
    for input, target in dataloader:
        # ...
        loss.backward()
        mean_loss += loss
    mean_loss /= len(dataloader)
```

Since PyTorch 0.4, `loss` is a 0-dimensional `Tensor`, which means that the addition to `mean_loss` keeps around the gradient history of each `loss`. The additional memory use will linger until `mean_loss` goes out of scope, which could be much later than intended. In particular, if you run evaluation during training after each epoch, you could get out of memory errors when trying to allocate GPU memory for the testing samples.

## 3. Use a closure in training

If you're coming from other languages, the Python scope of variables may not be what you are used to. In particular, variables declared in loops and conditional statements linger around for the surrounding scope -- they go out-of-scope only after the function that they are in terminates. For a more in-depth look, see [here](http://nbviewer.jupyter.org/github/rasbt/python_reference/blob/master/tutorials/scope_resolution_legb_rule.ipynb).

To avoid keeping Tensors allocated after they are no longer needed, one may declare a [closure](https://www.programiz.com/python-programming/closure). This assures us that any variables declared inside the closure go out of scope when the closure terminates. A practical example:

```python
    for input, target in dataloader:
        def handle_batch():
            x, y = input.to(device), output.to(device)
            pred = model(x)
            loss = loss_fn(pred, y)
            loss.backward()
            optimizer.step()

        handle_batch()
        # All of the variables defined above are now out of scope!
        # On CPU, they are already deallocated. On GPU, they will be deallocated soon.

    # Make sure deallocation has taken place
    if torch.cuda.is_available():
        torch.cuda.synchronize()
```

Since communication between your Python code and the GPU is asynchronous, the memory reserved by the closure might not have been deallocated right after training halts. To make sure this happens, one may call [`torch.cuda.synchronize()`](https://pytorch.org/docs/stable/_modules/torch/cuda.html#synchronize) before allocating more memory. This is useful if you are running testing or validation code after each epoch, to avoid Out Of Memory errors.

## 4. No more `Variable`-wrapping!

In earlier versions of PyTorch it was required to wrap Tensors in Variables to make them differentiable. This is what I mean:

```python
for x, y in training_loader:
    x, y = Variable(x), Variable(y)
    # ...
```

Since PyTorch 0.4.0 however, `Tensor` is merged with `Variable`, so now to allocate a `Tensor` that requires gradients to be computed with respect to it, one can simply do

```python
torch.tensor([1.0, 2.0], requires_grad=True)
```

Similarly, the above training loop doesn't require the explict `Variable()` calls, since the data in this case does not require grad, and gradients will flow through Tensors.

## 5. Model eval mode and `torch.no_grad()`
When evaluating, one can save memory and time by not tracking history during the forward pass for gradient calculation. This is easy to achieve with the `torch.no_grad()` context manager. 

Another thing good to remember is that during training, BatchNorm and other similar layers may be tracking batch / other statistics, that are not meant to be used during inference. The same holds for Dropout - usually one wants to apply dropout only during training. These layers can be switched to eval mode by calling `model.eval()`, and back to training mode using `model.train()`.

Given these, here's an example of a typical testing setup:

```python
with torch.no_grad():
    model.eval()

    # Testing code
    # ...
```

## More?

If you have any others to contribute, drop me an email!