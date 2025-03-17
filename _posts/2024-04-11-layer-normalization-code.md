---
layout: post
title: "layernorm code walk (llm, gpt2, laynorm, math)"
author: "melon"
date: 2024-04-11 22:05
categories: "2024"
tags:
  - llm
  - machine learning
---

@[gpt2][1] picked up the same architecture as the @[transformer][2], but the position of the
@[layernorm][3] was moved into the pre-normalization version.

the residual path of the transformer is kept clean, and the layernorms are now the first
layer of each block of the transformer, which positively improves training stability.

however, it's not easy to codewalk the formular from the @[pytorch layernorm][4],
because it's buried 30 layers deep behind an inscrutable dynamical dispatcher,
in possibly auto-generated cuda code (@[pytorch laynorm c implementation][5], @[pytorch laynorm cuda kernel function][6]).

<hr>

### # layernorm in the math form
the layernorm inference typically lie in following form:

$$
\text{LayerNorm}(x) = w \odot \frac{x - \mu}{\sqrt{\sigma^2 + \epsilon}} + b
$$

where $\odot$ is elementwise multiplication, $\mu$ is the mean, $\sigma^2$ is the variance,
and $\epsilon$ is a small constant to avoid division by zero.

$$
y/out = \frac{x-\mathrm{E}[x]}{\sqrt{\operatorname{Var}[x]+\epsilon}} * \gamma+\beta
$$

$$
norm = \frac{x-\mathrm{E}[x]}{\sqrt{\operatorname{Var}[x]+\epsilon}}
$$

$$
rstd\ (reciprocal\ standard\ deviation) = \frac{1}{\sqrt{\operatorname{Var}[x]+\epsilon}}
$$

$$
var = \frac{1}{N} \cdot \sum_i^N (x_i-\mathrm{E}[x])^2
$$

<hr>

### # layernorm in pytorch with autograd
pytorch can do the backward pass of layernorm forward layer by its @[autograd][7].

the magic of pytorch autograd is that after we call backward,
it will populate the grad attribute of all the tensors that have requires_grad=True
with the gradients of the loss with respect to that tensor.

these gradients are telling us the slope of the loss for all of the input numbers in
x, w, b. thus, the shape of x.grad, w.grad, and b.grad are exactly the same as
the shape of x, w, and b.

<hr>

### # layernorm implementation in pytorch with bare fist
the following will implement a layernorm manually using basic pytorch operations.
it is a lot less efficient than forwarding the pytorch layernorm module,
but it is algorithmically instructive.

```text
import torch
eps = 1e-5

class LayerNorm:
    """
    x is the input activations, of shape B,T,C
    w are the weights, of shape C
    b are the biases, of shape C
    """
    @staticmethod
    def forward(x, w, b):
        B, T, C = x.size()
        mean = x.sum(-1, keepdim=True) / C                                            # mean: B,T,C -> B,T,1
        xshift = x - mean
        var = (xshift**2).sum(-1, keepdim=True) / C                                   # variance: B,T,1
        rstd = (var + eps) ** -0.5                                                    # rstd: B,T,1, **-0.5 is 1/sqrt
        norm = xshift * rstd                                                          # norm: B,T,C
        out = norm * w + b                                                            # out: B,T,C
        cache = (x, w, mean, rstd)
        return out, cache

    @staticmethod
    def backward(dout, cache):
        x, w, mean, rstd = cache
        norm = (x - mean) * rstd
        dw = (dout * norm).sum((0, 1))                                                # dl/dw: bp blog eq 15
        db = dout.sum((0, 1))                                                         # dl/db: bp blog eq 16
        dnorm = dout * w                                                              # dl/dnorm = f(dl/dout)
        dx = dnorm - dnorm.mean(-1, keepdim=True) - norm * (dnorm * norm).mean(-1, keepdim=True)
        dx *= rstd                                                                    # dl/dx = f(dl/dnorm)
        return dx, dw, db

if __name__ == '__main__':
    B = 2
    T = 3
    C = 4

    x = torch.randn(B, T, C, requires_grad=True)                           # forward pass -> out, cache
    w = torch.randn(C, requires_grad=True)
    b = torch.randn(C, requires_grad=True)
    out, cache = LayerNorm.forward(x, w, b)

    dout = torch.randn(B, T, C)                                            # self-implemented backward pass
    dx, dw, db = LayerNorm.backward(dout, cache)
    print(dx)
    
    fakeloss = (out * dout).sum()                                          # autograd backward pass: dl/dout*out=loss
    fakeloss.backward()
    
    print("dx error:", (x.grad - dx).abs().max().item())                   # comparison
    print("dw error:", (w.grad - dw).abs().max().item())
    print("db error:", (b.grad - db).abs().max().item())
    
    x, w, mean, rstd = cache                                               # for reference checking in c also
    
    def write(tensor, handle):
        handle.write(tensor.detach().numpy().astype("float32").tobytes())
    
    with open('ln.bin', 'wb') as file:                                     # write state & computed tensors to bin file
        write(x, file)                                                     # (B, T, C)
        write(w, file)                                                     # (C, )
        write(b, file)                                                     # (C, )
        write(out, file)                                                   # (B, T, C)
        write(mean, file)                                                  # (B, T)
        write(rstd, file)                                                  # (B, T)
        write(dout, file)                                                  # (B, T, C)
        write(dx, file)                                                    # (B, T, C)
        write(dw, file)                                                    # (C, )
        write(db, file)                                                    # (C, )
```

<hr>

### # math principles of the back-propagation of layernorm in pytorch code
1 why $\partial loss / \partial norm$ can be computed as follows?

$$
dnorm = dout * w
$$

given dout, the following equation established:

$$
out = norm \cdot w + b
$$

by which the chain rule can be applied as:

$$
\frac{\partial loss}{\partial norm} = \frac{\partial loss}{\partial out} \cdot \frac{\partial out}{\partial norm}
= dout \cdot \frac{\partial out}{\partial norm} = dout \cdot w
$$

thus the code got proved.

<p style="margin-bottom: 20px;"></p>

2 why $\partial loss / \partial x$ can be computed as follows?

$$
dx = dnorm - dnorm.mean(-1, keepdim=True) - norm * (dnorm * norm).mean(-1, keepdim=True)
$$

given dout, with chain rule of back-propagation rule applied:

$$
\frac{\partial loss}{\partial x} = \frac{\partial loss}{\partial norm} \cdot
\frac{\partial norm}{\partial x}
$$

remind the relationship between norm and x, the above eq can be simplified as:

$$
\frac{\partial loss}{\partial x} = \frac{\partial loss}{\partial norm} \cdot
\frac{\partial ((x-\mathrm{E}[x]) \cdot rstd)}{\partial x}
$$

solve the math above according to (uv)'=u'v+v'u:

$$
dx = dnorm \cdot [\frac{\partial (x-\mathrm{E}[x])}{\partial x} \cdot
rstd + \frac{\partial rstd}{\partial x} \cdot (x-\mathrm{E}[x])]
$$

for simplicity, derive the partial derivative to certain item x_i rather than matrix form x:

$$
dx = dnorm \cdot \left(\frac{\partial (x-\frac{1}{N}\sum_{i=1}^N x_i)}{\partial x} \cdot
rstd + \frac{\partial [(\operatorname{Var}[x]+\epsilon)^{-\frac{1}{2}}]}{\partial x} \cdot
(x-\mathrm{E}[x])\right) \\

= dnorm \cdot \left( (1-\frac{1}{N}) \cdot rstd -
\frac{1}{2} (\operatorname{Var}[x]+\epsilon)^{-\frac{3}{2}} \cdot
\frac{\partial (\operatorname{Var}[x]+\epsilon)}{\partial x} \cdot
(x-\mathrm{E}[x])\right) \\

= dnorm \cdot \left( (1-\frac{1}{N}) \cdot rstd -
\frac{1}{2} (\operatorname{Var}[x]+\epsilon)^{-\frac{3}{2}} \cdot
\frac{\partial [\frac{1}{N} \sum_i^N(x_i-\mathrm{E}[x])^2]}{\partial x} \cdot
(x-\mathrm{E}[x])\right) \\

= dnorm \cdot \left( (1-\frac{1}{N}) \cdot rstd -
(\operatorname{Var}[x]+\epsilon)^{-\frac{3}{2}} \cdot 
\frac{x_i - \mathrm{E}[x]}{N} \frac{\partial (x_i - \mathrm{E}[x])}{\partial x} \cdot
(x-\mathrm{E}[x])\right) \\

= dnorm \cdot \left( (1-\frac{1}{N}) \cdot rstd -
(\operatorname{Var}[x]+\epsilon)^{-\frac{1}{2}} \cdot 
\frac{x_i - \mathrm{E}[x]}{\sqrt {\operatorname{Var}[x]+\epsilon}} \cdot
\frac{1}{N} (1-\frac{1}{N}) \cdot
\frac{x-\mathrm{E}[x]}{\sqrt {\operatorname{Var}[x]+\epsilon}} \right)
$$

recall the definition of rstd, norm, the above equation can be simplified as:

$$
dx = dnorm \cdot \left( (1-\frac{1}{N}) \cdot rstd -
rstd \cdot
norm \cdot
\frac{1}{N} (1-\frac{1}{N}) \cdot
norm \right) \\

= rstd \cdot
\left( (dnorm-\frac{dnorm}{N}) -
norm \cdot
\frac{1}{N} (\frac{N-1}{N}) \cdot dnorm \cdot norm \right) \\

= rstd \cdot
\left( (dnorm-\frac{dnorm}{N}) -
norm \cdot
((N-1) \cdot \frac{dnorm}{N} \cdot \frac{norm}{N} \right)
$$

finally, the equation to compute $\partial loss / \partial x$ got proved:

$$
dx = rstd \cdot \left( dnorm-dnorm.mean(-1, keepdim=true)
- norm \cdot (N-1) \cdot (dnorm \cdot norm).mean(-1, keepdim=true) \right)
$$

<hr>

### # tradeoff between memory & computation when do back-propagation
there's no strict rules to save mean and rstd during forwarding, cause they can be recomputed
in backward pass.
the scale of mean & rstd are very small with shape of (B,T), however, norm is
of shape (B,T,C), and the gap in dimension helps determine whether to compute or store.

the tradeoff is very common cross all layers, thus different implementations of network layers
in deep learning frameworks may have different checkpoint settings.

afterall, the tradeoff has nothing todo with final models results, cause they varies only 
in intermediate state of forward & backward process.

<hr>

### # from pytorch to c: how to manage tensor memory model in c
this part will move from pytorch to c implementation and get rid of tensor abstraction.

a brief introduction to tensors:  
1 a 1d block of memory called storage that holds the raw data.  
2 a view over that storage that holds its shape.
see @[pytorch internals][8] for more details.

so for example given a 3d tensor:

```text
torch.manual_seed(42)
B, T, C = 2, 3, 4
a = torch.randn(B, T, C)
print(a)

tensor([[[ 1.9269,  1.4873,  0.9007, -2.1055],
         [ 0.6784, -1.2345, -0.0431, -1.6047],
         [ 0.3559, -0.6866, -0.4934,  0.2415]],

        [[-1.1109,  0.0915, -2.3169, -0.2168],
         [-0.3097, -0.3957,  0.8034, -0.6216],
         [-0.5920, -0.0631, -0.8286,  0.3309]]])
```

which in a tensor with shape (2,3,4), but the underlying memory for storation is just
a single 1d array of size 2\*3\*4=24.
when try index this tensor at a[1,2,3], the pytorch framework will computes the offset in
the 1d arrary as 1\*3\*4 + 2\*4 + 3 = 23, and return array[23].

the general method to get value of any element with index b,t,c for a tensor with shape (B,T,C):

```text
tensor[b,t,c] = array[b*T*C + t*C + c]
```

confirm the above conclusion in pytorch:

```text
b,t,c = 1,2,3
print(a[b,t,c])                       # 0.3309
print(a.view(-1)[b*T*C + t*C + c])    # 0.3309
```

for gpt2 input shape as (batch_size, tensor, channel), inc tensor offset by 1
is equivalent to traverse the channel dimension.

<hr>

### # layernorm implementation in c 
layernorm.c:
```text
#include <stdio.h>
#include <stdlib.h>
#include <math.h>

void layernorm_forward(float* out, float* mean, float* rstd,
                       float* inp, float* weight, float* bias,
                       int B, int T, int C){
    float eps = 1e-5f;
    for(int b = 0; b < B; b++){
        for(int t = 0; t < T; t++){
            float* x = inp + b * T * C + t * C;            // inp: start pos of input [b,t,:], x[i]=inp[b,t,i]
            float m = 0.0f;                                // calculate mean across channel idx
            for(int i = 0; i < C; i++){
                m += x[i];
            }
            m = m/C;
            float v = 0.0f;                                // calculate variance (without bias correction)
            for(int i = 0; i < C; i++){
                float xshift = x[i] - m;
                v += xshift * xshift;
            }
            v = v/C;
            float s = 1.0f / sqrtf(v + eps);               // calculate rstd
            float* out_bt = out + b * T * C + t * C;       // write forward res to out[b,t,:]
            for(int i = 0; i < C; i++){
                float n = (s * (x[i] - m));                // normalized output
                float o = n * weight[i] + bias[i];         // scale and shift it
                out_bt[i] = o;                             // write
            }
            mean[b * T + t] = m;                           // cache mean for bp
            rstd[b * T + t] = s;                           // cache rstd for bp
        }
    }
}

void layernorm_backward(float* dinp, float* dweight, float* dbias,
                        float* dout, float* inp, float* weight, float* mean, float* rstd,
                        int B, int T, int C){
    for(int b = 0; b < B; b++){
        for(int t = 0; t < T; t++){
            float* dout_bt = dout + b * T * C + t * C;             // dl/dout at pos dout[b,t,:]
            float* dinp_bt = dinp + b * T * C + t * C;             // dl/dx at pos dinp[b,t,:]
            float* inp_bt = inp + b * T * C + t * C;               // cache x
            float mean_bt = mean[b * T + t];                       // cache mean
            float rstd_bt = rstd[b * T + t];                       // cache rstd

            float dnorm_mean = 0.0f;                               // first: two reduce operations
            float dnorm_norm_mean = 0.0f;
            for(int i = 0; i < C; i++){
                float norm_bti = (inp_bt[i] - mean_bt) * rstd_bt;  // compute norm again rather than using cache
                float dnorm_i = weight[i] * dout_bt[i];            // dnorm
                dnorm_mean += dnorm_i;                             // dnorm.mean
                dnorm_norm_mean += dnorm_i * norm_bti;             // dnorm.(dnorm.mean)
            }
            dnorm_mean = dnorm_mean / C;
            dnorm_norm_mean = dnorm_norm_mean / C;
            
            for(int i = 0; i < C; i++){                            // now iterate again and accumulate all the gradients
                float norm_bti = (inp_bt[i] - mean_bt) * rstd_bt;
                float dnorm_i = weight[i] * dout_bt[i];
                
                dbias[i] += dout_bt[i];                            // db: dl/dbias
                dweight[i] += norm_bti * dout_bt[i];               // dw: dl/dweight 
                float dval = 0.0f;                                 // init dx: gradient contribution to input
                dval += dnorm_i;                                   // dx = dnorm
                dval -= dnorm_mean;                                // dx - dnorm.mean
                dval -= norm_bti * dnorm_norm_mean;                // norm * (dnorm*norm).mean(-1,)
                dval *= rstd_bt;                                   // dx *= rstd
                dinp_bt[i] += dval;
            }
        }
    }
}

int check_tensor(float* a, float* b, int n, char* label){          // poor man's tensor checker
    int ok = 1;
    printf("%s\n", label);
    for(int i = 0; i < n; i++){
        if(fabs(a[i] - b[i]) <= 1e-5){
            printf("OK ");
        }
        else{
            printf("NOT OK ");
            ok = 0;
        }
        printf("%f %f\n", a[i], b[i]);
    }
    return ok;
}

int main(){
    int B = 2;                                                     // batch
    int T = 3;                                                     // time sequence length
    int C = 4;                                                     // number of channels

    float* x = (float*) malloc(B * T * C * sizeof(float));
    float* w = (float*) malloc(C * sizeof(float));
    float* b = (float*) malloc(C * sizeof(float));
    float* out = (float*) malloc(B * T * C * sizeof(float));
    float* mean = (float*) malloc(B * T * sizeof(float));
    float* rstd = (float*) malloc(B * T * sizeof(float));
    float* dout = (float*) malloc(B * T * C * sizeof(float));
    float* dx = (float*) malloc(B * T * C * sizeof(float));
    float* dw = (float*) malloc(C * sizeof(float));
    float* db = (float*) malloc(C * sizeof(float));

    FILE *file = fopen("ln.bin", "rb");                            // read reference information generated by pytorch
    if(file == NULL){
        printf("Error opening file\n");
        return 1;
    }
    fread(x, sizeof(float), B * T * C, file);
    fread(w, sizeof(float), C, file);
    fread(b, sizeof(float), C, file);
    fread(out, sizeof(float), B * T * C, file);
    fread(mean, sizeof(float), B * T, file);
    fread(rstd, sizeof(float), B * T, file);
    fread(dout, sizeof(float), B * T * C, file);
    fread(dx, sizeof(float), B * T * C, file);
    fread(dw, sizeof(float), C, file);
    fread(db, sizeof(float), C, file);
    fclose(file);

    float* c_out = (float*) malloc(B * T * C * sizeof(float));     // c implementation for layernorm
    float* c_mean = (float*) malloc(B * T * sizeof(float));
    float* c_rstd = (float*) malloc(B * T * sizeof(float));
    layernorm_forward(c_out, c_mean, c_rstd, x, w, b, B, T, C);    // forward pass

    check_tensor(out, c_out, B*T*C, "out");                        // check correctness of forward pass
    check_tensor(mean, c_mean, B*T, "mean");
    check_tensor(rstd, c_rstd, B*T, "rstd");

    float* c_dx = (float*) calloc(B * T * C, sizeof(float));       // backward pass (calloc init grads as 0)
    float* c_dw = (float*) calloc(B * T, sizeof(float));
    float* c_db = (float*) calloc(B * T, sizeof(float));
    layernorm_backward(c_dx, c_dw, c_db, dout, x, w, c_mean, c_rstd, B, T, C);

    check_tensor(c_dx, dx, B*T*C, "dx");                           // check correctness of backward pass
    check_tensor(c_dw, dw, C, "dw");
    check_tensor(c_db, db, C, "db");

    free(x); free(w); free(b);                                     // resource release
    free(out);
    free(mean); free(rstd);
    free(dout);
    free(dx); free(dw); free(db);
    return 0;
}
```

1 bp variable updates code style  
an important detail of above implementation to note is that: i.e take dinp_bt for consideration,
always update the final bp variable with += operations, but never use = or use *=.

it's an important rule because if you have one bp variable used multiple times in a graph,
the bp gradients always stay in the addup format.
in forward/bp implementation for gpt2, this rule is not strict because there's no
exotic computation branch, but a proper way for both situations.

so during training, first do zero_grad to set all the gradients to zero,
and then accumulate into them during backward pass.

<p style="margin-bottom: 20px;"></p>

2 layernorm vs rmsnorm: the difference  
unlike gpt2, @[llama2.c][9] replace layernorm with the simpler rmsnorm, the implementation is as:

```text
void rmsnorm(float* o, float* x, float* weight, int size){
    float ss = 0.0f;                     // calculate sum of squares
    for(int j = 0; j < size; j++){
        ss += x[j] * x[j];
    }
    ss /= size;
    ss += 1e-5f;
    ss = 1.0f / sqrtf(ss);
    for(int j = 0; j < size; j++){       // normalize and scale
        o[j] = weight[j] * (ss * x[j]);
    }
}
```

how does rmsnorm differ to layernorm?  
1 rmsnorm does not subtract the mean, it only normalizes by the norm (not std norm), which is very trendy
because it works just as well.
whatmore, the rmsnorm does not have biases, only has a weight for scaling the norm, in contrast, gpt2 use
too many biases everywhere and somehow can be removed.

the network itself can simulate biases if it needs them, e.g. by allocating extra one channel dimensions
to be constant, and then weight multiplying the constant dimension will effectively work like a bias,
which will significantly simplify the code.

2 the llama2 inference code has no batch dimension (batch size = 1).
you could have batched inference as well, especially when host an llm to accept many simultaneous
queries a meantime.

while, if you're just running an llm locally, a single stream of generation is enough.
so there's no need for parallelism to support multiple streams at once.
thus, llama2.c is not batched, therefore you won't see any loops that look like:

```text
for(int b = 0; b < B; b++){...}
```

3 the rmsnorm test/product inference code has no time dimension T.
during training, we can loop over time inside each layer and calculate norm at all time steps.
however, during test/product inference, which need generate one token at a time, then feed the token
predicted at time t into the forward pass of the transformer at the next time step t+1.

so there's no no loops in forwarding stage like:

```text
for(int t = 0; t < T; t++){...}
```

but the loop over time dimension does exist, ref: @[llama2#timeloop][10],
in which it lies outside of the transformer forward pass.

during llama2 inference, there's no track of any intermediate calculations, memory, or cache.
because during test/product inference, there is no backward pass to follow.
as a result, the memory consumption of test/product inference is significantly lower than that of training.

<hr>

### # code execution for pytorch.py & layernorm.c
as a result of all these difference, training gpt2 is significantly more complex and involved
than llama2, both algorithmically and computationally.

write out the reference data using pytorch firstly:
```text
$ python layernorm.py
```

then compile and execute the c version to validate the details of inference & bp:

```text
$ gcc layernorm.c -o layernorm -lm            // link math
$ gcc -std=c99 layernorm.c -o layernorm -lm   // use c99 std to avoid error: for loop initial declarations
$ ./layernorm
```

everything between the two flavor matches ok.

<hr>

### # autograd of layernorm by our own autograd engine
todo (conclude in another blog)



[1]: https://d4mucfpksywv.cloudfront.net/better-language-models/language_models_are_unsupervised_multitask_learners.pdf
[2]: https://arxiv.org/abs/1706.03762
[3]: https://arxiv.org/abs/1607.06450
[4]: https://pytorch.org/docs/stable/generated/torch.nn.LayerNorm.html
[5]: https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/native/layer_norm.cpp
[6]: https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/native/cuda/layer_norm_kernel.cu
[7]: https://www.youtube.com/watch?v=VMj-3S1tku0
[8]: http://blog.ezyang.com/2019/05/pytorch-internals/
[9]: https://github.com/karpathy/llama2.c
[10]: https://github.com/karpathy/llama2.c/blob/master/run.c#L747
