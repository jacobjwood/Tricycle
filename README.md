# Tricycle
<p align="center">
    <img width="223" alt="tricycle_logo" src="https://github.com/bclarkson-code/Tricycle/assets/57139598/62405944-b27b-49bc-93c3-17ba93fc8ad7">
</p>

Tricycle is a fast, minimal, fully functional deep learning library written from scratch using only python and numpy.

The file `train_smol_gpt.py` trains a 49M parameter, GPT-2 style language model that can produce passable python code in ~2 days on a single RTX 3090.

The entire library, from the automatic differentiation engine to a GPT, is written in ~4500 lines of python + numpy code.

Using [CuPY](https://cupy.dev/), all Tricycle code can run on either a Cuda-capable GPU or a CPU.


## Table of Contents

- [Installation](#installation)
    - [CPU Installation](#cpu-installation)
- [Training a GPT on shakespeare](#training-a-gpt-on-shakespeare)
- [How it works](#how-it-works)
    - [Automatic Differentiation](#automatic-differentiation)
    - [Einsum](#einsum)
      - [Summing along an axis](#summing-along-an-axis)
      - [Sum of an entire tensor](#sum-of-an-entire-tensor)
      - [Transpose](#transpose)
      - [Matrix multiplication](#matrix-multiplication)
    - [Building a simple neural network](#building-a-simple-neural-network)
    - [Optimisations](#optimisations)
      - [Batching](#batching)
      - [GPU](#gpu)
      - [Fusing](#fusing)
      - [Other optimisations](#other-optimisations)
        - [Inplace tensor updates](#inplace-tensor-updates)
        - [Mathematical optimisations](#mathematical-optimisations)
        - [Hardware optimisations](#hardware-optimisations)
- [Contact](#contact)

## Installation
Tricycle uses [conda](https://docs.conda.io/en/latest/) to manage dependencies. While we do support CPU-only computation, optimisation efforts have been focussed on GPU computation so it is pretty slow. If you do have a CUDA capable GPU I would strongly recommend installing the gpu version of Tricycle.

If you have a CUDA capable GPU, you can install Tricycle as follows.
```bash
conda env create -f environment.yml -n tricycle
conda activate tricycle
```

<details>
    <summary>CPU and test installation</summary>
If you want to install test dependencies you can do the following.

```bash
conda env create -f environment.test.yml -n tricycle
conda activate tricycle
```

### CPU Installation
If you want to install Tricycle for CPU, you can do the following.
```bash
conda env create -f environment.cpu.yml -n tricycle
conda activate tricycle
```

If you want to install test dependencies on CPU you can do the following.
```bash
conda env create -f environment.cpu.test.yml -n tricycle
conda activate tricycle
```
</details>


## Training a GPT on Shakespeare
The following toy script will train a small GPT to generate convincing Shakespeare.
On my RTX 3090, this takes ~30 mins. For a more realistic training script with metric tracking, gradient accumulation, a validation dataset etc, take a look at `train_smol_gpt.py`

```python
import pickle

from tqdm import tqdm

from tricycle.configs import ShakespeareConfig
from tricycle.dataset import CausalLMDataset
from tricycle.loss import CrossEntropy
from tricycle.models import GPT
from tricycle.optimisers import AdamW
from tricycle_datasets.shakespeare import Shakespeare

config = ShakespeareConfig()
model = GPT(config)

tokens = Shakespeare(vocab_size=config.vocab_size)
dataset = (
    CausalLMDataset(
        tokens=tokens,
        vocab_size=config.vocab_size,
        batch_size=config.batch_size,
        context_window=config.context_window,
    )
    .batch()
    .shuffle()
    .to_tensor()
)
loss_fn = CrossEntropy()
optimiser = AdamW(
    learning_rate=config.max_learning_rate,
    weight_decay=config.weight_decay,
    betas=(config.beta1, config.beta2),
)

model.to_gpu()
loading_bar = tqdm(range(config.steps))
for step in loading_bar:
    optimiser.step()
    inputs, outputs = next(dataset)
    inputs = inputs.to_gpu()
    outputs = outputs.to_gpu()

    logits = model(inputs)
    loss = loss_fn(outputs, logits)
    loss.backward()

    loading_bar.set_description(f"loss: {loss:.3f}")
    model.update(optimiser)

# save results
with open("model.pkl", "wb") as f:
    pickle.dump(model, f)
```
Once trained, you can generate infinite shakespeare plays as follows:

```bash
python inference.py model.pkl
```

## How it works
Tricycle code centers around objects called `Tensors`. A `Tensor` is a wrapper around a numpy array that adds some extra features:

```python
from tricycle.tensor import to_tensor

tensor = to_tensor([1,2,3])
print(tensor) # Output: Tensor([1. 2. 3.])
```

You can do a lot of things with a tensor

```python
from tricycle.functions import Softmax

a = to_tensor([1,2,3])
b = to_tensor([4,5,6])

# addition
print(a + b) # Output: Tensor([5. 7. 9.], name=badd)

# comparison
print(a < b) # Output: Tensor([ True  True  True])

# more complex functions
print(Softmax()(a)) # Output: Tensor([0.09003057 0.24472848 0.66524094], name=softmax)

```

### Automatic Differentiation
Unlike vanilla numpy, every operation in Tricycle is attached to a derivative.
When you do some operations on your `Tensor`, Tricycle keeps track of what you did and allows you to differentiate the output.

```python
x = to_tensor(2)

y = x ** 2 + 3 * x + 4
print(y) # Output: Tensor(14.0, name=+ 4)

# derivative of y with respect to (wrt) x is
# 2 * x + 3 = 7
y.backward() # differentiate wrt y
print(x.grad) # Output: Tensor(7.0)
```

This works on multidimensional tensors

```python
import numpy as np

shape = (6,5,4,3,2)
a = to_tensor(np.random.random(shape))
b = to_tensor(np.random.random(shape))

c = a * b # elementwise multiply

c.backward() # differentiate wrt c
assert a.grad.close_to(b) # derivative of c wrt a is b
assert b.grad.close_to(a) # derivative of c wrt b is a
```

And even works through complex operations like attention

```python
from tricycle.blocks import MultiHeadSelfAttention

attention = MultiHeadSelfAttention(
    embedding_dim=32,
    n_heads=2,
    context_window=32,
)

# batch_size, n_tokens, embedding_dim
shape = (4,32,32)
input = to_tensor(np.ones(shape), is_batched=True)

output = attention(input)
output.backward() # differentiate wrt output

print(input.grad) # Output: Tensor([[[ 2.5441039  -2.0558214  -1.7923143  ...
assert input.grad.shape == (4,32,32)
```

When you run an operation (`Op`), the output has two pieces of information attached:
 - `args`: The inputs to the function
 - `back_fns`: The functions that should be executed to calculate the derivative wrt each of the inputs

Surprisingly, this all that you need to perform automatic differentiation on an arbitrarily complicated sequence of `Op`s.
Because we keep track of the `args` for each operation, we can start at the output of a set of `Op`s and traverse through them to reach every input to the sequence: the operations form a tree.

Thanks to the [chain rule](https://en.wikipedia.org/wiki/Chain_rule), if we apply each `back_fn` that we pass through on our way through the tree, when we get to an input, we will have calculated the derivative of the output wrt the input.
Despite implementing it myself, I still feel like this couldn't possibly work, and yet it does!


The entirety of the algorithm can be found in [`tensor.py`](https://github.com/bclarkson-code/Tricycle/blob/main/src/tricycle/tensor.py#L145).

It ends up being a topological sort to figure out which order to traverse the tree and then a simple traversal, applying the `back_fns` along the way.

If you want a more detailed explanation, I've talked about it on [my blog](https://bclarkson-code.com/posts/llm-from-scratch-scalar-autograd/post.html).

### Einsum

Tricycle makes use of (in my opinion underutilised) einsum operations.
Einsum is a generalisation of a large number of matrix operations.

You can use it by assigning each axis in your matrices a letter of the
alphabet (called an index). You can define the operation you want to perform
by simply listing the indices you want in your inputs and output, separated by
an arrow.

For example, you can define the transpose of a 2d tensor as follows:

```python
from tricycle.einsum import Einsum

a = to_tensor([[1,2],[3,4]])
print(Einsum("ij->ji")(a)) # Output: Tensor([[1. 3.], [2. 4.]], name=einsum ij->ji)
```

Here, we use einsum to swap indices i and j: a transpose.

There are only two rules to remember with einsum:
 - If an index does not appear in the output, any inputs that contain it
   will be summed along that axis:
    ```python
    print(Einsum("ij->i")(a)) # Tensor([3. 7.], name=einsum ij->i)
    ```

 - If an index appears in more than one input, the tensors will be multiplied
   along that axis

    ```python
    b = to_tensor([[5,6],[7,8])
    print(Einsum("ij,jk->ik")(a,b)) # Tensor([[19. 22.], [43. 50.]], name=einsum ij,jk->ik)
    ```

For example:
#### Summing along an axis

https://github.com/bclarkson-code/Tricycle/assets/57139598/c575c958-19ed-4406-8a1b-d2390663ba96

#### Sum of an entire tensor

https://github.com/bclarkson-code/Tricycle/assets/57139598/efbb5eaa-656c-40e5-a32d-b0f5e7bd28f5

#### Transpose

https://github.com/bclarkson-code/Tricycle/assets/57139598/f8b35a6b-f102-44f1-a7cd-b6b2e765f275

#### Matrix multiplication

https://github.com/bclarkson-code/Tricycle/assets/57139598/1ed18428-11de-4990-a0f4-12d1310d6898

Becuase every `Op` in Tricycle needs a derivative, we need to figure out what the
derivative of `Einsum` is. Thankfully, if you sit down and go through the
maths (index notation is really helpful here) you'll find that you can follow
these two, really simple rules to differentiate an einsum operation wrt a
given input:

 - Swap the indices for the input and output
 - Replace the original input with your current derivative

For example, the derivative of a transpose works like this:

```python
# forward operation
y = Einsum('ij->ji')(a)

# swap the input with the current grad (a grid of ones in this case)
grad = to_tensor(np.ones_like(y))

# swap the indices
derivative = Einsum('ji->ij')(grad)
```

And for a more complex operation (a dense layer on a 4d input) like this:

```python
# forward operation
input = to_tensor(np.random.random((5, 4, 3, 2)))
weights = to_tensor(np.random.random((3,6)))
y = Einsum('zxTb,bW->zxTW')(inputs, weights)

grad = to_tensor(np.ones_like(y))

# swap the indices + replace inputs
derivative = Einsum('zxTb,zxTW->bW')(inputs, grad)
```

This little trick significantly simplifies code, as well as reducing the
amount of maths I had to do to implement different operations.

### Building a simple neural network

Einsum and an automatic differentiation engine are all we need to build a simple neural network. Lets try to train a model on the [iris dataset](https://scikit-learn.org/stable/auto_examples/datasets/plot_iris_dataset.html)
We can start with a [`Dense` Layer](https://github.com/bclarkson-code/Tricycle/blob/main/src/tricycle/layers.py#L34).

```python
from tricycle.layers import Dense

x = to_tensor([1,2,3])
layer = Dense(from_size=3, to_size=1)

print(layer(x)) # Output: Tensor([-2.238703], name=dense)
```

Next, neural networks need a nonlinearity (otherwise they reduce to expensive linear regressions).

Tricycle has a few [nonlinearities](https://github.com/bclarkson-code/Tricycle/blob/main/src/tricycle/activation.py) (also called activation functions). Here we can choose the simplest: `ReLU`.


```python
from tricycle.activation import ReLU

x = to_tensor([-1, 0, 1])
activation_fn = ReLU()

print(activation_fn(x)) # Output: Tensor([0. 0. 1.], name=> 0)
```

We also need a loss function. We're predicting a category so we can use CrossEntropy

```python
from tricycle.loss import CrossEntropy

label = to_tensor([0, 1, 2], dtype=int)
predicted = to_tensor([[0,0,1], [0,0,1], [0,0,1]])
loss = CrossEntropy()

print(loss(label, predicted)) # Output: Tensor(1.2181114, name=cross_entropy)
```

Finally, we need an optimiser to update our weights. We can use [Stochastic Gradient Descent](https://github.com/bclarkson-code/Tricycle/blob/main/src/tricycle/optimsers.py#L14).
In Tricycle, you can use an optimiser the weights of a model as follows:

```python
from tricycle.activation import ReLU
from tricycle.layers import Dense, Sequential
from tricycle.optimisers import StochasticGradientDescent

# build a model
layer_1 = Dense(4, 16)
layer_2 = Dense(16, 3)
relu = ReLU()
model = Sequential(layer_1, relu, layer_2)

# create an optimiser
optimiser = StochasticGradientDescent(learning_rate=1e-1)

# do a forward and backward pass
x = to_tensor([1,2,3,4])
out = model(x)
out.backward()

# update the weights
model.update(optimiser)
```

We can put all of this together to train a simple neural network on the iris
dataset.

```python
import numpy as np
from sklearn.datasets import load_iris

from tricycle.activation import ReLU
from tricycle.tensor import to_tensor
from tricycle.layers import Dense, Sequential
from tricycle.loss import CrossEntropy
from tricycle.optimisers import StochasticGradientDescent

LEARNING_RATE = 1e-1
N_STEPS = 1000

np.random.seed(42)
X, y = load_iris(return_X_y=True)
inputs = to_tensor(X, is_batched=True)

# The class labels need to be ints for crossentropy
outputs = to_tensor(y, is_batched=True, dtype=int)

# create a model
layer_1 = Dense(4, 16)
layer_2 = Dense(16, 3)
relu = ReLU()
model = Sequential(layer_1, relu, layer_2)

loss_fn = CrossEntropy()
optimiser = StochasticGradientDescent(learning_rate=LEARNING_RATE)

for step in range(N_STEPS):
    y_pred = model(inputs)
    loss = loss_fn(outputs, y_pred)
    if step == 0:
        print(f"Initial loss: {loss}") # Output: Initial loss: Tensor(3.974701, name=cross_entropy)

    loss.backward()
    model.update(optimiser)

print(f"Final loss: {loss}") # Output: Final loss: Tensor(0.08622341, name=cross_entropy)

# Calculate accuracy
predicted_labels = np.argmax(y_pred.array, axis=-1)
accuracy = (predicted_labels == outputs.array).mean()
print(f"Accuracy: {accuracy:.2f}") # Output: Accuracy: 0.97
```

### Optimisations

Deep learning is famously computationally heavy. If we want to train anything
in a reasonable amount of time, there are several optimisations we need to make.

#### Batching
The first, and arguably most important, optimisation is batching. Instead of
applying operations to each input individually, if we are clever about how we design
an operation, we can apply an operation to multiple operations at once.

For example, suppose we are multiplying a batch of tensors by a weight matrix.
We could do it like this:

```python
# batch of 1024 64x64 tensors
inputs = to_tensor(np.ones((1024, 64, 64)))
weights = to_tensor(np.random.random((64,64)))

output = [Einsum('ij,jk->ik')(inp, weights) for inp in inputs]
# 62.2 ms ± 186 μs per loop (mean ± std. dev. of 7 runs, 10 loops each)
```

But we can use the properties of `Einsum` to do the same thing like this

```python
output = Einsum('aij,jk->aik')(inputs, weights)
# 29.1 ms ± 99.2 μs per loop (mean ± std. dev. of 7 runs, 10 loops each)
```

Which is more than 2x faster.

Some `Op`s in tricycle behave slightly differenly, depending on
whether a tensor batched or not. You can tell tricycle to use the batched
version of `Op`s for a tensor by simply calling `.to_batched`. To convert it
back, you can call `.from_batched`.

#### GPU
As well as batching, another improvement that has a big impact on performance
is using a GPU. For this, we can use a library called [CuPY](https://cupy.dev/).
CuPY lets you run numpy code on a GPU. This means that we can use the same code
for CPU as well as GPU computation which greatly simplifies the codebase (
and avoids me needing to write CUDA kernels, for now).

Every tensor in tricycle has an `.xp` method. By default, this is just the
numpy library:

```
import numpy as np

tensor = to_tensor([1,2,3])

assert tensor.xp == np
```

But if you call `.to_gpu` on a tensor, this is the cupy library:

```
import cupy as cp

tensor = to_tensor([1,2,3])

tensor.to_gpu()

assert tensor.xp == cp
```

(`xp` stands for `np` or `cp` because x is an "unknown"). This is really handy
because it lets us write functions like this:

```python
def forward(self, tensor: Tensor):
    """
    Apply softmax. The softmax is only applied to the final
    dimension of the tensor
    Note: the tensor is normalised for numeric stability
    """
    xp = tensor.xp

    exp = xp.exp(
        # subtract the largest value for numeric stability
        tensor.array - xp.max(tensor.array, axis=-1, keepdims=True)
    )
    denominator = xp.sum(exp, axis=-1, keepdims=True)
    self._out = exp / denominator

    result = to_tensor(self._out)
    result.args = (tensor,)
    result.name = "softmax"
    result.is_batched = tensor.is_batched
    result.back_fns = (self.backward,)

    return result
```

Becuase cupy has the same interface as numpy, this function will automatically
run on the right device, with no code changes.

#### Fusing

One of the problems I faced when trying to use Tricycle is that it used up
a lot more memory than I expected. Because the `args` and `back_fns` need to
be stored for every `Op`, a lot of memory was being used to store intermediate
values.

For more operations like `Softmax`, this quickly adds up. However,
we can avoid a lot of this overhead by pre-computing the combined derivative.
In the case of `Softmax` (see above), we could have built it entirely out of
low level Tricycle operations and this does work. When you sit down and work
out the derivative for softmax manually, it turns out to be pretty simple:

```python
def backward(self, grad: Tensor) -> Tensor:
    xp = grad.xp

    inner = xp.sum(grad.array * self._out, axis=-1, keepdims=True)
    self._grad = self._out * (grad.array - inner)
    return to_tensor(
        self._grad,
        is_batched=grad.is_batched,
        requires_grad=grad.requires_grad,
    )
```

This kind of operation is a very common optimisation technique in deep learning
called 'Operator Fusing'. This ends up being a big optimisation for tricycle
because it lets us replace operations like `MultiHeadSelfAttention`, which
would usually have 10s of intermediate values, with a single `forward` and
`backward` function with a minimal set of intermediate values.

#### Other optimisations
While batching, using a GPU and fusing are the major optimisations, I'd like
to provide some honorable mentions.

##### Inplace tensor updates
While probably obvious to many readers, updating tensors in-place rather than
replacing them with a new tensor caused a big speedup.

##### Mathematical optimisations
Operations like `CrossEntropy` can be implemented by applying a softmax and then
applying the crossentropy operation but, if you do a bit of algebra,
you can do something called the `log-sum-exp` trick to simplify the expression
and cut down on the computations needed.

##### Hardware optimisations
As mentioned above, the GPU computation was performed on an NVIDIA RTX 3090.
Understandably, this gets quite hot when training (probably something to do with
it being in my cupboard?) which can reduce performance due to thermal
throttling. However, I found that by removing my computer case and placing
a household fan on top, I get about 30% better performance.

![IMG_0713](https://github.com/bclarkson-code/Tricycle/assets/57139598/958f12b4-caaa-4f2a-b9d0-2f5a7fc1e5a5)

Putting all of these things together, Tricycle can train a small language model on shakespeare in ~30 mins. Andrej Karpathy can [do this in pytorch](https://github.com/karpathy/nanoGPT/tree/master) in around 7 minutes on my machine (with a like-for-like config) which, given that the entire Tricycle project is in python, means that Tricycle is surprisingly fast. That said, more work is needed to get the speed up.


## Contact
Want to work together? You can reach me at: [bclarkson-code@proton.me](mailto:bclarkson-code@proton.me)
