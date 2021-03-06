George
======

Fast Gaussian processes for regression
--------------------------------------

This is *one-dimensional* GP code built on top of Sivaram Ambikasaran's
`HODLR solver <https://github.com/sivaramambikasaran/HODLR_Solver>`_
designed to be fast on **huge** problems. So far it's only being used to
model the noise in *Kepler* light curves but it should be more generally
useful.

The code is mainly written in (mostly undocumented) C++ with Python bindings.
You should probably stick with the Python for now.

Installation
------------

**Latest stable release**

You'll first need to install `Eigen 3 <http://eigen.tuxfamily.org/>`_ and
(obviously) `NumPy <http://www.numpy.org/>`_. Then, for the default
installation, just run::

  pip install george

If you have installed Eigen in a strange place, you'll need to install from
the git repository.

**Development version**

To install the development version, first clone the repository and initialize
the submodules::

  git clone https://github.com/dfm/george
  cd george
  git submodule init
  git submodule update

Then, install the package using::

  python setup.py install

If you've installed Eigen in a strange place, you can specify this by running::

  python setup.py install --eigen-include=/path/to/eigen

Usage
-----

Here's the simplest possible example of how you might want to use George::

  import numpy as np
  import george
  from george.kernels import ExpSquaredKernel

  # Generate some fake noisy data.
  x = 10 * np.sort(np.random.rand(10))
  yerr = 0.2 * np.ones_like(x)
  y = np.sin(x) + yerr * np.random.randn(len(x))

  # Set up the Gaussian process.
  kernel = ExpSquaredKernel(1.0)
  gp = george.GaussianProcess(kernel)

  # Pre-compute the factorization of the matrix.
  gp.compute(x, yerr)

  # Compute the log likelihood.
  print(gp.lnlikelihood(y))

  # Draw 100 samples from the predictive conditional distribution.
  t = np.linspace(0, 10, 500)
  samples = gp.sample_conditional(y, t, size=100)

This should result in a distribution like the following:

.. image:: https://raw.github.com/dfm/george/master/images/demo.png

**A drop-in replacement for your likelihood function**

Let's imagine that you're currently fitting your data assuming uncorrelated
Gaussian uncertainties. In this case, you probably have a likelihood function
in your code that looks something like::

  x, y, yerr = ...

  def lnlike(params):
      m = compute_model(params, x)
      return -0.5 * np.sum((y - m) ** 2 / yerr**2)

To use George as a replacement for this likelihood function, you would just
do::

  kernel = a * ExpSquaredKernel(s)
  gp = george.GaussianProcess(kernel)
  gp.compute(x, yerr)

  def lnlike(params):
      m = compute_model(params, x)
      return gp.lnlikelihood(y - m)

If you also want to model or marginalize over the (hyper-)parameters of the
GP, this would be replaced by::

  def lnlike(lna, lns, params):
      kernel = np.exp(lna) * ExpSquaredKernel(np.exp(lns))
      gp = george.GaussianProcess(kernel)
      gp.compute(x, yerr)

      m = compute_model(params, x)
      return gp.lnlikelihood(y - m)

This model will (of course) be much slower because the covariance matrix
will need to be recomputed and factorized at every step.

Finally, if you want to include a *jitter* parameter (a factor that accounts
for underestimated error bars), it is as simple as::

  def lnlike(lna, lns, lnjitter, params):
      kernel = np.exp(lna) * ExpSquaredKernel(np.exp(lns))
      gp = george.GaussianProcess(kernel)
      j2 = np.exp(2*lnjitter)
      gp.compute(x, np.sqrt(yerr**2 + j2))

      m = compute_model(params, x)
      return gp.lnlikelihood(y - m)

**More sophisticated kernel models**

The kernels in George need to be written in C++ but it comes with a few
pre-loaded and an expressive model building syntax. For example, if you have
both high and low frequency noise, you could model it as a mixture of kernels::

  from george.kernels import ExpSquaredKernel
  kernel = ExpSquaredKernel(3.0) + 0.5 * ExpSquaredKernel(0.1)

If the noise is periodic or quasi-periodic, you might try something like a
damped harmonic oscillator::

  from george.kernels import Matern32Kernel, CosineKernel
  kernel = 1e-3 * Matern32Kernel(1.0) * CosineKernel(0.5)

To be specific, the following kernels are defined:

* ``ExpKernel(s) = exp(-fabs(r/s))``
* ``ExpSquaredKernel(s) = exp(-0.5*(r/s)**2)``
* ``CosineKernel(P) = cos(2*pi*r/P)``
* ``Matern32Kernel(s) = (1+sqrt(3)*r/s) * exp(sqrt(3)*r/s)``

The following figure (generated by `examples/simple.py
<https://github.com/dfm/george/blob/master/examples/simple.py>`_) shows draws
from a few different kernels:

.. image:: https://raw.github.com/dfm/george/master/demo.png

License
-------

George is being developed by `Dan Foreman-Mackey <http://dfm.io>`_ and the
source is available under the terms of the `MIT license
<https://github.com/dfm/george/blob/master/LICENSE>`_.

Copyright 2012-2014 Dan Foreman-Mackey
