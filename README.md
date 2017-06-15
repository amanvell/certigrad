# Certigrad

Certigrad is a proof-of-concept of a new way to develop machine learning systems, in which the following components are developed simultaneously:

* The implementation itself.
* A library of background mathematics.
* A formal specification of what the implementation needs to do in terms of the mathematics.
* A machine-checkable proof that the implementation satisfies its specification.

Specifically, Certigrad is a system for optimizing over stochastic computation graphs, that we debugged systematically in the Lean Theorem Prover, and ultimately proved correct in terms of the underlying mathematics.

## Warning

We are currently in the process of porting Certigrad to the newest version of Lean, and parts of it are in an inconsistent state. Please check back in a week.

## Background: stochastic computation graphs

Stochastic computation graphs extend the computation graphs that underly systems like TensorFlow and Theano by allowing nodes to represent random variables and by defining the loss function to be the expected value of the sum of the leaf nodes over all the random choices in the graph. Certigrad allows users to construct arbitrary stochastic computation graphs out of the primitives that we provide. The main purpose of the system is to take a program describing a stochastic computation graph and to run a randomized algorithm (stochastic backpropagation) that, in expectation, samples the gradients of the loss function with respect to the parameters.

## Correctness

#### Stochastic backpropagation

Here is the theorem stating and proving that our implementation of stochastic backpropagation is correct:

https://github.com/dselsam/certigrad/blob/master/src/certigrad/backprop_correct.lean#L14-L28

Informally, it says that for any stochastic computation graph, `backprop` computes a vector of tensors such that each element of the vector is a random variable that (in expectation) equals the gradient of the expected loss of the graph with respect to that parameter.

Even more informally, `∇ E[loss(graph)] = E[backprop(graph)]`.

#### Certified optimizations

We also implemented two stochastic-computation-graph-transformations, one to "reparameterize" a graph so that a random variable no longer depends directly on a parameter, and one to integrate out the KL-divergence of the multivariate isotropic Gaussian.

https://github.com/dselsam/certigrad/blob/master/src/certigrad/kl.lean#L79-L90

https://github.com/dselsam/certigrad/blob/master/src/certigrad/reparam.lean#L70-L79

#### Verifying specific graphs

Finally, we prove that stochastic backpropagation is correct on a specific stochastic computation graph that results from applying the two transformations mentioned above. The resulting graph represents the autoencoding variational Bayes model. Specifically, we prove that all the necessary preconditions for the certified transformations and the stochastic backpropagation algorithm hold.

TODO(dhs): link to theorem statement

#### Formal proof

In the process of proving a theorem, Lean constructs a formal proof certificate that can be automatically verified by a small stand-alone executable, whose soundness is based on a well-established meta-theoretic argument embedding the core logic of Lean into set theory, and whose implementation has been heavily scrutinized by many developers.  Thus no human needs to be able to understand why a proof is correct in order to trust that it is.

#### Impurities

We have adopted a very high standard for our proofs, but there are a few ways in which Certigrad falls short of the purist ideal.

* We axiomatize the background mathematics instead of constructing it from first principles.
* By necessity, we execute with floating-point numbers even though our correctness theorems only hold in terms of infinite-precision real numbers.
* For performance, we replace the primitive tensor operations with calls to Eigen at runtime.
* We execute in a virtual machine, which is not designed to be as trustworthy as the proof-checker for the core logic.

#### Todo

Lean 3 is not backwards compatible with Lean 2 and some basic lemmas (e.g. properties of lists) are still being ported to the newest version. As placeholders, we currently rely on a few such lemmas without proof. We are also in the process of restruring our tactic programs to use the simplifier more aggressively, and the proof that all the preconditions hold for the AEVB model is not currently working.

## Performance

Provable correctness need not come at the expense of computational efficiency: proofs need only be checked once and they introduce no ongoing costs or runtime overhead. Although the algorithms we verify in this work lack many optimizations, most of the time training machine learning systems is spent multiplying matrices, and we are able to acheive competitive performance simply by linking with an optimized library for matrix operations (Eigen). As an experiment, we trained an Auto-Encoding Variational Bayes (AEVB) model on MNIST using ADAM and find that our performance is competitive with TensorFlow (on CPUs).

## Benefits of the methodology

Although our methodology introduces many new challenges, it offers substantial benefits as well.

#### Debugging

First, our methodology provides a way to debug machine learning systems systematically.

Implementation errors can be extremely difficult to detect in machine learning systems---let alone to localize and address---since there are many other potential causes of undesired behavior. For example, an implementation error may lead to incorrect gradients and so cause a learning algorithm to stall, but such a symptom may also be caused by noise in the training data, a poor choice of model, an unfavorable optimization landscape, an inadequate search strategy, or numerical instability. These other issues are so common that it is often assumed that any undesired behavior is caused by one of them. As a result, actual implementation errors can persist indefinitely without detection. Errors are even more difficult to detect in stochastic programs, since some errors may only distort the distributions of random variables and may require writing custom statistical tests to detect.

With our methodology, the formal specification can be used to test and debug machine learning systems exhaustively at a _logical_ level without needing to resort to empirical testing at all. The process of proving that the specification holds will expose all implementation errors, oversights and hidden assumptions. Once it has been proved, every interested party can be certain that the implementation is correct without needing to trust any human involved or to understand how the program works.

#### Synthesis

Second, our methodology can enable some parts of the implementation to be synthesized semi-automatically.

Whereas with the status-quo methodology, the compiler has _no idea_ what the program is supposed to do and so can only catch superficial syntactic errors, with our methodology the theorem prover knows _exactly_ what the program is supposed to do, and can provide much more useful assistance accordingly. As a simple example, suppose we want to compile a 2-layer MLP into a single primitive operator to avoid the overhead of graph-processing at runtime. Normally this would involve deriving the gradient of the compound function by hand. However, since in our methodology the theorem prover knows about the underlying mathematics, including the relevant gradient rules and the algebraic properties of tensors, it can assist in the derivation of the gradient for the new operator.

The possibilities for synthesis go way beyond simply automating algebraic derivations. When developing Certigrad, we started proving that the specification held before even implementing the difficult parts of the system, and used the resulting proof-obligations to help determine what the program needed to do. The formal specification, and ultimately the machine-checkable proof of correctness, allowed us to implement the system correctly without a coherent, global understanding of _why_ the system was correct. Indeed, we mostly relegated that burden to the computer.

#### Aggressive optimizations

Third, our methodology may enable safely automating much more aggressive transformations than would otherwise be advisable. For example, one could write a procedure that searches for components of a stochastic computation graph that can be integrated out analytically, that makes use of large libraries of integral identities as well as procedural methods that are impossible for humans to simulate by hand. Such a procedure may be able to acheive super-human variance reduction on many models yet may be extremely difficult to implement reliably; if the procedure is able to generate a machine-checkable certificate for a given transformation, the transformation can be trusted regardless of the complexity of the procedure itself.

#### Documentation

Fourth, a formal specification (even without a formal proof) can serve as precise documentation for a system, and can make it much easier to understand what various parts of the code do, what preconditions are assumed to hold at various places, and what invariants are being maintained. Such precise documentation can be useful for any software system but can be especially useful for machine learning systems, since not all developers may have the necessary mathematical expertise to fill in the gaps of informal descriptions.

## Incrementality

Our methodology may already be economical for high-assurrance systems, and yet there is still a lot of work to be done to make it practical for mainstream developments for which correctness is only "optional". However, a crucial aspect of our methodology is that it can be adopted _incrementally_. One can write only a little bit of the code in Lean and simply wrap and axiomatize the rest (as we did with Eigen). One can also write down shallow correctness properties, and only prove that a few of these properties hold. We hope that over time, as the tools mature, developers will find it worth the cost to pursue our methodology further and will be able to reap more of its benefits.

## Building Certigrad

TODO(dhs): use new Lean package manager to minimize dependencies

TODO(dhs): use forthcoming Lean foreign function interface instead of forking Lean.

TODO(dhs): include command that downloads Eigen to a subdirectory

## Warning

We have formally proved that Certigrad is correct (modulo the impurities mentioned above), **but this does not imply that Certigrad does what you expect**. All it means is that the theorems linked to above are true, given the assumptions. Here is an example of a "gotcha" that we left in to stress the point: the `gemv` operator computes an incorrect gradient. If a user writes a stochastic computation graph in which a `gemv` operator needs to be differentiated through, Certigrad will compute the wrong result. How can this undesired behavior not contradict the theorem that the gradients are always correct? The main correctness theorem (https://github.com/dselsam/certigrad/blob/master/backprop_correct.lean#L14-L25) requires the assumption that the preconditions of each of the operators that need to be differentiable are satisfied. The precondition of the `gemv` operator is `false` and so can never be satisfied. Note that the graph-specific theorem (the analogue of https://github.com/dselsam/certigrad/blob/master/aevb_theorems.lean#L201-L208) will not be provable, exactly because the precondition of `gemv` cannot be established. In other words, the math never said that the gradients would be correct on that particular graph. Note that this particular issue could be avoided by requiring the precondition for each operator to be true whenever the function it computes is differentiable.

There are also many kinds of possible errors for which Certigrad has no way of protecting you from. For example, if you write a stochastic computation graph but you accidently enter `-` instead of `+` somewhere, Certigrad will take your instructions at face value and optimize over the graph you gave it. That said, Certigrad can mitigate this particular problem in some cases by allowing you to write a simpler version of the desired graph, and then prove that your more complicated version is equivalent to the simpler one. Such a proof would not go through unless both graphs had the same exact error.

Lastly, Certigrad is designed to be a _proof-of-concept_, not a production system. There are many features that would need to be added to make it useful as an artifact. Over the course of development, we encountered many problems that will need to be solved to make our methodology economical, and we are much more interested in addressing those challenges than in extending and maintaining Certigrad as an end in itself.

## Team

* Daniel Selsam, Stanford University
* Percy Liang, Stanford University
* David Dill, Stanford University

## Acknowledgments

This work was supported by Future of Life Institute grant 2016-158712.