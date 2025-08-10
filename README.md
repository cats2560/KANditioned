## KANditioned: Fast, Generalizable Training of KANs via Lookup Interpolation and Proximal Gradient

<!-- # Fast and generalizable training of Kolmogorov-Arnold Network (KAN) via look up tables and prefix-sum reparameterization.  -->

<!-- ### TL;DR: This KAN implementation uses a linear spline with uniformly spaced control points and reparameterizes from a B-spline to a cumulative ReLU-based spline for better optimization conditioning. Training is accelerated using a parallel prefix-sum and fast interpolation via parameter lookup between the two nearest basis functions. -->

<!-- ## TL;DR: This KAN implementation achieves orders-of-magnitude faster training, improved conditioning, and better generalization by reparameterizing linear B-splines into cumulative ReLU-based splines and leveraging parallel scan and lookup-based interpolation. -->

<!-- ## TL;DR: This KAN implementation achieves orders-of-magnitude faster training and better generalization by reparameterizing linear B-splines into cumulative ReLU-based splines and leveraging parallel scan and lookup-based interpolation. -->

<!-- TL;DR: We achieved orders-of-magnitude faster training and improved conditioning by reparameterizing linear B-splines into cumulative ReLU splines. Training is accelerated via leveraging parallel scan for reparameterization and fast interpolation via parameter lookup between two nearest control points. -->

TL;DR: Training is accelerated by orders of magnitude through exploiting the structure of linear (C⁰) spline with uniformly spaced control points, where spline(x) can be calculated as a linear interpolation between the two nearest control points. This is in constrast with the typical summation often seen in B-spline, reducing the amount of computation required. Optimization ill-conditioning is mitigated through regularizing/reparameterizing by applying a Gaussian smoothing kernel on the parameters using FFT convolution. This has a time complexity of O(N log N), with a depth of O(log N), where N is the number of parameters. This technique is inspired by Batch Normalization, where both employ a differentiable transformation to improve problem conditioning, albeit this technique is applied to parameters rather than activations. Although the time complexity of DCT is O(N log N), with a depth of O(log N), this is independent of batch size, making the proximal step negligible as the batch size or number of control points increases. 

<!-- ## TL;DR: This KAN implementation improves training speed by orders of magnitude, along with better conditioning and generalization, by reparameterizing linear B-splines into cumulative ReLU-based splines and using parallel scan and lookup-based interpolation. -->
 
<!-- ## TL;DR: This KAN implementation achieves orders of magnitude faster training and better generalization by reparameterizing B-splines into cumulative ReLU splines and leveraging parallel scan and lookup-based interpolation. -->

<!-- ## TL;DR: We speed up KAN training by orders of magnitude and improve generalization using a better spline formulation and fast interpolation trick. -->

<!-- #### TL;DR: We accelerate KAN training by orders of magnitude and improve generalization by reparameterizing the linear spline with parallel scan and using a fast interpolation trick. -->

### How This Works

This implementation of KAN uses a linear (C⁰) spline, with uniformly spaced control points (see Figure 1).

> **Figure 1.** Linear B-spline example:  
> ![Linear B-spline example](image-1.png)

To improve the conditioning of the optimization problem, the spline is reparameterized from the B-spline basis as proposed in the original paper (see Equation 1), which has strictly local support, to a cumulative ReLU spline formulation (see Equation 2). In this formulation, each parameter contributes via a ReLU term with support extending in one direction from its associated breakpoint b<sub>l</sub>, yielding semi-global, rather than local, influence. As each parameter update causes semi-global changes in the spline shape, this biases learning towards simpler, more generalizable structure, as opposed to fragile, local representations, while preserving the same theoretical expressitivity.

> **Equation 1.** B-spline formula:
>
> <img style="height: 50px" alt="B-spline Formula" src="image.png">



> **Equation 2.** ReLU-spline formula:
>
> <img style="height: 50px" alt="ReLU-spline Formula" src="image-2.png">

This reparameterization is implemented via a parallel scan (prefix sum) with O(log N) time complexity and O(N) work complexity, where N is the number of parameters, independent of batch size. For more details, see Equation 2. Training speed was further improved by orders of magnitude by exploiting the fact that under the uniformly spaced control points with linear basis spline formulation, spline(x) can be efficiently evaluated by calculating the index of the two nearest control points, gather, and linearly interpolating between them, rather than summing over all basis functions. At a certain point, scaling the number of control points do not cause any noticeable increase in computation time, as most of the time is spent waiting for the parameter gather, which is still significantly more efficient than summing over all basis functions. 

<!-- This implementation of KAN uses a linear (C⁰) spline, with uniformly spaced control points (see Figure 1). To improve the conditioning of the optimization problem, the spline is reparameterized from the B-spline basis as proposed in the original paper (see Equation 1), which has strictly local support, to a cumulative ReLU-based spline formulation (see Equation 2). In this formulation, each parameter contributes via a ReLU term with support extending in one direction from its associated breakpoint b<sub>l</sub>, yielding semi-global, rather than local, influence. This reparameterization is implemented via a parallel scan (prefix sum) with O(log N) time complexity, where N is the number of parameters, independent of batch size. Training speed was further improved by orders of magnitude by exploiting the fact that under the linear basis spline formulation, spline(x) can be efficiently calculated by looking up the parameters of the two nearest linear bases and linearly interpolating between them, rather than summing over all basis functions. -->

**(1)** Training of ReLU-based spline with 1k parameters:


https://github.com/user-attachments/assets/032e686f-a08d-48f3-bfcc-11fb53eca21e


<!-- <video src="training_animation_conditioned.mp4" controls></video> -->

**(2)** Training of linear B-spline with 1k parameters:


https://github.com/user-attachments/assets/3488606a-5d8f-4c03-aa7e-64356eb9bf37


<!-- <video src="training_animation_ill_conditioned.mp4" controls></video> -->

<!-- instead of optimizing the spline under the B-spline parameterization (see Equation 1), it is reparameterized and directly optimized under the [insert ReLU formulation], where each parameter now affects the spline globally, rather than having strictly local support. The reparameterization is done using parallel scan (prefix sum) with O(log N) time complexity, where N is the number of parameters, independent of batch size. The number of computations can be further reduced by recognizing that under the linear basis spline formulation, spline(x) can be efficiently calculated by looking up the parameters of the two nearest linear bases and linearly interpolating between them. -->

<!-- <img style="height: 60px" alt="B-spline Formula" src="https://latex.codecogs.com/png.image?\dpi{400}\bg_white\large\displaystyle S(x)%20=%20\sum_{i=0}^{n}%20c_i%20B_{i,k}(x)%20\qquad%20B_{i,0}(x)%20=%20\begin{cases}1%20&%20\text{if%20}%20t_i%20\leq%20x%20%3C%20t_{i+1}\\0%20&%20\text{otherwise}\end{cases}%20\qquad%20B_{i,k}(x)%20=%20\frac{x%20-%20t_i}{t_{i+k}%20-%20t_i}%20B_{i,k-1}(x)%20+%20\frac{t_{i+k+1}%20-%20x}{t_{i+k+1}%20-%20t_{i+1}}%20B_{i+1,k-1}(x)"> -->

<!-- <img style="height: 60px" alt="ReLU-spline Formula" src="https://latex.codecogs.com/png.image?\dpi{400}\bg_white\large\displaystyle%20S(x)%20=%20\sum_{\ell=0}^{N-1}%20\left[%20w^+_{\ell}%20\cdot%20ReLU(x%20-%20b_{\ell})%20+%20w^-_{\ell}%20\cdot%20ReLU(b_{\ell}%20-%20x)%20\right]"> -->

### Short Random Thoughts
- KAN suffers from training difficulties, which includes the training speed and the ill-conditioning of solution space
- Local support leads to ill-conditioning and exploration of high complexity solutions
- ReLU-based spline with its semi-global support, although is just theoretically as expressive as B-spline, biases learning towards simpler and more global, generalizable structures before exploring more complex solutions while B-splines do not have that property when overparameterized. Or, in other words, learning functions progressively, increasing in complexity from very simple form initially.

### TODO:
- Add installation pip package and figures showing the visualize interp tensor as well as the example
<!-- - Update with appropriate EMA for min and max -->
- Add baselines comparing current spline implementation, B-spline, and MLP with same number of parameters
<!-- - Add ReLU spline figure -->
<!-- - Add gifs or videos showing the training process with and without reparameterization -->
- Add gifs or videos showing the training process
<!-- - Add the steps to go from ReLU-spline formulation to prefix-sum -->
- Revisit the writing to be more polished later
- Write optimized Triton kernel
- Put up Google Colab Notebooks
- Add a warning about needing appropriate learning rate, batch norm, and initialization. Recommended to put a range at -2 to 2
- Add a line about exploiting the structure in KAN
- Add something about Gaussian kernel smoothing and discrete heat/diffusion equation
- Add a line about batchnorm and spline range
- Add a line about provable convergence of proximal gradient
- Check difference in torch.compile between nn.Embedding and F.embedding
- Check difference between feature-major order versus batch-major order
- Add a license (probably MIT)
- Add a line about using torch.compile
- Add baselines on backward and forward passes
- Add a note on this can scale with number of control points very easily