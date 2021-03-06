# CEED benchmarks with deal.II

This project provides small programs to evaluate the ceed benchmark cases
http://ceed.exascaleproject.org/bps with the matrix-free evaluation routines
provided by the deal.II finite element library,
https://github.com/dealii/dealii

### Results for mass operator (BP1)

As a first experiment, we run the mass matrix operator within a conjugate
gradient iterative solver with the identity preconditioner and record the time
for one solver iteration, i.e., we run the solver until convergence to a
tolerance of 1e-6 and divide the total solver time by the number of
iterations.  On SuperMUC Phase 1 (2x8 cores Sandy Bridge E5-2680 2.7 GHz), the
following results are obtained:

![alt text](https://github.com/kronbichler/ceed_benchmarks_dealii/blob/master/bp1/gnuplot/node1.png)
![alt text](https://github.com/kronbichler/ceed_benchmarks_dealii/blob/master/bp1/gnuplot/node256.png)

When we turn to a more detailed analysis of the behavior at p=6 and q=8, we
get the following data:
![alt text](https://github.com/kronbichler/ceed_benchmarks_dealii/blob/master/bp1/gnuplot/p6_q8.png)

One solver iteration gets slower on large computations in the left part of the
plot, which is mainly due to the MPI communication in the two inner products per
iteration in the CG solver. However, when looking specifically at the time per
matrix-vector product, we also see that it is 2x faster than the CG
iterations. In other words, the three vector updates and two inner products in
CG represent half the compute time in the CG solver. Again, this is not too
surprising since the kernel is mostly memory bandwidth bound.

Finally, the time for the matrix-vector product on a newer architecture,
namely one node of Intel Broadwell E5-2690 v4 (2x14 cores at 2.6 GHz nominal
frequency, Turbo Boost to 2.9 GHz enabled) is as follows:
![alt text](https://github.com/kronbichler/ceed_benchmarks_dealii/blob/master/bp1/gnuplot/matvec_bdw.png)

The timings for this system are better then before. For those sizes that fit
into the L3 cache, the performance is more than twice as high at 2.77 Gdofs/s
versus 1.11 Gdofs/s on 16 cores of Sandy Bridge, reflecting the increased
arithmetic power of 28 cores of a newer generation that includes fused
multiply-add (FMA) instructions. However, at around 10^7 dofs the throughput
is only higher by 65% (1.4 Gdofs/s vs 883 Mdofs/s) because of the memory
bandwidth limitations.

### Results for Laplace operator (BP3)

We now turn to the evaluation of the three-dimensional Laplacian on a general
geometry (bake-off problem 3). Here, we only look at the details of the
matrix-vector product and not the conjugate gradient solver as a whole,
because the vector operations (that again take a signicant proportion of the
time for `p>2`) are the same as before. On one node of Intel Broadwell E5-2690
v4 (2x14 cores at 2.6 GHz nominal frequency, Turbo Boost to 2.9 GHz enabled)
the following throughput is reached:
![alt text](https://github.com/kronbichler/ceed_benchmarks_dealii/blob/master/bp3/gnuplot/matvec_bdw.png)

As opposed to the mass matrix where the only sensible option is to read the
precomputed Jacobian determinant from memory, there are several options to
implement the variable coefficients for the Laplacian. In the figure above, we
used the final geometry tensor, i.e. we store the 6 independent components of
the symmetric matrix `det(J) J^{-1} J^{-T}` and load it in each quadrature
point. As for the mass matrix, a clear peak in throughput is visible when the
all data fits into the level 3 cache. Since this is six times more data than
for the mass matrix, the high performance is limited to significantly smaller
problem sizes (and won't be visible at all once we have many nodes with MPI
due to the communication overhead).

In the next figure we have looked at some of the alternatives. One alternative
we have considered is the default option in deal.II targeting generic
differential operators (rather than the specialized code for the Laplacian)
where both the Jacobian matrix `J^{-T}` and the determinant `det(J)` are
stored for each quadrature point and accessed from memory during operator
evaluation. Since this option does both 5/3 more memory transfers and slightly
more computations, the throughput is lower. Higher throughputs can be recorded
when the geometry is computed on the fly. In the case *linear geometry on the
fly*, the Jacobians of the geometry are evaluated from the vertex positions
(or rather, a combination of the vertex positions in terms of the factors in
the reference coordinates, to simplify the evaluation of the Jacobian on the
fly) of the cells in a trilinear way. This variant is slower than *merged
coefficient tensor* for small sizes when the latter fits into the 70 MB of L3
cache, but it is faster for larger sizes because it significantly reduces the
memory transfer. To be more specific, the linear geometry case is compute
bound (i.e., bound by the instruction throughput without significant phases
where the execution waits for data from memory to arrive). In case a curved
representation of elements is desired, the linear geometry approximation is of
course inadequate and another strategy must be used. One option to compute the
Jacobian on the fly is represented by *q-geometry on the fly* that evaluates
the Jacobian from the precomputed location of quadrature points in real space
using a collocation derivative. This variant uses only half the memory
transfer of *merged coefficient tensor*, but it is compute bound rather than
memory bound on this particular system. Thus, the resuling performance is
slower than *merged coefficient tensor*, but still faster than *plain
Jacobian*. We refer to [Kronbichler and Kormann, 2017] for details on the
variants.

![alt text](https://github.com/kronbichler/ceed_benchmarks_dealii/blob/master/bp3/gnuplot/matvec_coefficients.png)

On the Broadwell system, we can enable hyperthreading and use 56 MPI ranks
rather than 28. Since our implementation is bound by the instruction
throughput with phases of vector access (gather/scatter) and computations (sum
factorization, geometry evaluation) interleaved, 2-way simultaneous
multithreading (hyperthreading in Intel language) can be used to improve
instruction flow. We record around 10-15% higher throughput with this
variant:
![alt text](https://github.com/kronbichler/ceed_benchmarks_dealii/blob/master/bp3/gnuplot/matvec_coefficients_ht.png)

Finally, we show the analysis of the four ways to represent the geometry also
for a higher polynomial degree of eight. The performance is very similar to
the case `p=5`.
![alt text](https://github.com/kronbichler/ceed_benchmarks_dealii/blob/master/bp3/gnuplot/matvec_coefficients_p8.png)

### Results for Laplace operator (BP5)

The next benchmark to evaluate is the BP5 problem, which represents a 3D
Laplacian with the integrals computed with the Gauss-Lobatto quadrature rule
rather than the Gauss quadrature, and using p+1 rather than p+2 quadrature
points. This setup comes along with a collocation between node points and
quadrature points. This allows to cut the work in the sum factorization code
paths (related to interpolation of values and gradients between the node
values and quadrature points) into half. On one node of Intel Broadwell
E5-2690 v4 (2x14 cores at 2.6 GHz nominal frequency, Turbo Boost to 2.9 GHz
enabled) the following throughput is reached for the full BP5 problem,
including the conjugate gradient solver and a Jacobi preconditioner on the
diagonal of the matrix:
![alt text](https://github.com/kronbichler/ceed_benchmarks_dealii/blob/master/bp5/gnuplot/bp5_bdw.png)

By looking only at the matrix-vector product, the numbers get considerably
higher, as for the BP1 and BP3 problems. Again, we spend more than 50% of time
on vector operations in the CG solver. Furthermore, we clearly see the effect
of the L3 cache.
![alt text](https://github.com/kronbichler/ceed_benchmarks_dealii/blob/master/bp5/gnuplot/matvec_bdw.png)

Next, we report results for a newer architecture, namely Intel Xeon Scalable
Platinum 8168 (Skylake) with 2x24 cores. The BP5 benchmark runs about 1.85
times (8.0e8 DoFs/sec/it on one node vs 4.3e8 DoFs/sec/it for *p*=7) as fast
for large sizes that run from main memory. Near the peak at 500,000 DoFs per
node, Skylake reaches 3.0e9 DoFs/s versus 1.3e9 on Broadwell. These values are
close to the capabilities of hardware, as we have measured up to 220 GB/s of
memory speed on Skylake versus 120 GB/s on Broadwell (1.83 times) and the
arithmetic peak on Skylake at 3840 GFlops/node versus Broadwell's 1300
GFlops/node (2.95 times, speedup was 2.31).
![alt text](https://github.com/kronbichler/ceed_benchmarks_dealii/blob/master/bp5/gnuplot/bp5_skx.png)

The matrix-vector product behaves similarly, again showing about a 2x boost by Skylake.
![alt text](https://github.com/kronbichler/ceed_benchmarks_dealii/blob/master/bp5/gnuplot/matvec_skx.png)

### Literature

* [M. Kronbichler and K. Kormann
  (2012)](https://doi.org/10.1016/j.compfluid.2012.04.012). A generic
  interface for parallel cell-based finite element operator
  application. *Comput. Fluids*, 63:135-147.

* [M. Kronbichler and K. Kormann
  (2017)](https://arxiv.org/abs/1711.03590). Fast matrix-free evaluation of
  discontinuous Galerkin finite element operators. *Preprint*
  arXiv:1711.03590.

### Prerequisites and installation

The benchmark problems are designed as small programs that run against a
compiled deal.II finite element library that in turn needs a C++ compiler
adhering to the C++11 standard, an MPI implementation, and cmake. The
following software packages are needed:

* deal.II, using at least version 8.5.0, see www.dealii.org. deal.II must be
  configured to also include the following external packages (no direct access
  to this packages is necessary, except for the interface through deal.II):

* MPI

* p4est for providing parallel adaptive mesh management on forests of
  quad-trees (2D) or oct-trees (3D). For obtaining p4est, see
  http://www.p4est.org. p4est of at least version 0.3.4.2 is needed for
  running this project. Installation of p4est can be done via a script
  provided by deal.II (the last argument specifies the desired installation
  directory for p4est, e.g. $HOME/sw/p4est):
```
/path/to/dealii/doc/external-libs/p4est-setup.sh p4est-1.1.tar.gz /path/to/p4est/install
```

Given these dependencies, the configuration of deal.II can be done
through the following script:
```
cmake \
    -D CMAKE_CXX_FLAGS="-march=native" \
    -D CMAKE_INSTALL_PREFIX="/path/to/dealii/install/" \
    -D DEAL_II_WITH_MPI="ON" \
    -D DEAL_II_WITH_LAPACK="ON" \
    -D DEAL_II_WITH_P4EST="ON" \
    -D P4EST_DIR="/path/to/p4est/install/" \
    ../deal.II
```

Since the matrix-free algorithms in deal.II make intensive use of advanced
processor instruction sets (e.g. vectorization through AVX or similar), it is
recommended to enable processor-specific optimizations (second line,
`-march=native`). The path on the third line specifies the desired
installation directory of deal.II, and the last line points to the location of
the source code of deal.II relative to the folder where the cmake script is
run. After configuration, run

```
make -j8
make install
```

to compile deal.II and install it in the given directory. After installation,
the deal.II source and build folder are no longer necessary (unless you find
bugs in deal.II and need to modify that code). It is also possible to build
the test cases against a build folder of deal.II (which is what the author of
this package does almost exclusively).
