Simple Fast Smooth Interpolation
================================

Original Author: Benjamin Dreyer

Quick Start
-----------

1. Include the header in your code. It has no dependencies (not even to the
	standard library), so including it anywhere should cause no issues. You may
	even wrap it in another namespace of your choice if you so please.
2. Use the interpolation function as follows:
	```c++
	shakatir::sfsi::interpolate<FLOAT, S1, ..., SN>(FUNCTION, X1, ..., XN)
	```
	The parameters are:
	- `FLOAT`: You probably want this to be `float`, `double` or `long double`,
		but any numeric type that behaves similar to a built-in floating point
		type will suffice. In particular the type has to be:
		- default-constructible, copy-constructible and copy-assignable,
		- constructible from `long long` via `static_cast<FLOAT>(...)`,
		- supporting binary arithmetic operators `+`, `-`, `*` and `/`.

		Results are likely not meaningful if the type does not model an
		approximation of the real numbers. The code will likely be slow if the
		resulting type is not trivially copyable and trivially destructible.
	- `S1, ..., SN`: The *smoothness parameters*. For each dimension you pass an
		individual template parameter that determines how smooth the
		interpolation in that dimension is supposed to be. The parameters are
		positive integer values and have the following interpolation behaviors:
		- `1`: Bilinear interpolation. Produces a zig-zag pattern by drawing a
			straight line between two ajacent points. Requires a neighborhood at
			indices `{0, 1}`.
		- `2`: Bicubic interpolation. Continuous in the first derivative.
			Requires an environment at indices `{-1, 0, 1, 2}`.
		- `3`: Continuous in the first and second derivative. This is likely the
			highest value you will need. Requires a neighborhood at	indices
			`{-2, -1, 0, 1, 2, 3}`.
		- `4` or higher: For the paranoid. Makes a *mathematically* smoother
			curve at the risk of having more extreme swings to the naked eye
			(the same risk that lies in all spline interpolations using
			polynomials of high degree). The highest supported value as of now
			is `10` as higher values would cause an overflow of the signed
			64-bit integers in the precomputed table.
		Generally a smoothness of `S` produces a function that is continuous in
		the first `S-1` derivatives and requires a neighborhood at indices
		`{1-S, 2-S, ..., S}`.
	- `FUNCTION`: a functor that when called with integer arguments returns a
		value implicitly convertible to `FLOAT`. The functor will be called as
		follows:
		```c++
		FUNCTION(i1, ..., iN)
		```
		`i1` is an integer in the environment of `S1`, `i2` lies in the
		environment of `S2` and so on. Each possible combination of indices will
		be used for a total of `(2 * S1) * (2 * S2) * ... * (2 * SN)` values in
		the `N`-dimensional environment.
	- `X1, ..., XN`: Coefficients of type `FLOAT` giving the position at which
		the interpolated value shall be calculated. For the purpose of
		interpolation, each value should lie within the range of 0 and 1, but if
		they fall outside that range, it will simply result in an
		*extrapolation* of the interpolation. In particular, small rounding
		errors in the input won't break the result.

Examples:
- 1D bilinear interpolation between two points:
	```c++
	float seed[2]{ ... };
	float x = ...;
	float result = interpolate<float, 1>([&seed](int i){ return seed[i]; }, x);
	```
- 2D bicubic interpolation of a 4-by-4 neighborhood:
	```c++
	double seed[4][4]{ ... };
	double x = ..., y = ...;
	double result = interpolate<double, 2, 2>([&seed](int i, int j){ return seed[i + 1][j + 1]; }, x, y);
	```
	Note how we add 1 to map the range `-1, .., 2` to the range `0, ..., 3`.
- 1D bicubic interpolation where the position can range over several integers:
	```c++
	float seed[10]{ ... };
	float x = ...;
	int j = static_cast<int>(x);
	float result = interpolate<float, 2>([&seed, j](int i){ return seed[i + j + 1]; }, x - j);
	```
	Note that the interpolation will be a smooth and continuous function over
	the entire interval of `0 <= x < 8`, but it will actually be joined together
	from 8 individual slices.

- 3D interpolation with mixed smoothness and procedural, cyclical neighborhood:
	```c++
	int (*hash)(int, int, int) = ...;
	double x = ..., y = ..., z = ...;
	int xi = (int)x, yi = (int)y, zi = (int)z;
	double result = interpolate<double, 2, 1, 3>(
		[hash, xi, yi, zi](int i, int j, int k){
			return (hash((xi + i) & 15, (yi + j) & 15, (zi + k) & 15) % 1025) / 1024.0.0;
		}, x - xi, y - yi, z - zi);
	```

What is this?
-------------

This library provides a family of interpolation functions that map an arbitrary
number of real numbers onto a single real value. They are related to the bicubic
interpolation known from image editing, however neither is a generalization over
the other.

The potential use case is interpolation of data that is known in a regular
pattern, for example:
- creating a landscape from a heightmap or from procedurally generated reference
	points,
- calculating intermediate positions of a smooth animation.

Features of this library are:
- Single-header, header-only: Just include it in your codefile in whatever way
	suits you best.
- Standard compliant: The library makes no use of platform dependent features
	like intrinsics. It relies on the compiler's ability to generate efficient
	assembly. Any standard compliant compiler with support for C++14 and later
	should be able to compile this file.
- Zero dependencies: This header includes no other headers---not even from the
	standard library.
- Low memory footprint: On modern x64 processors and with proper optimization,
	the stack won't even be touched at all for lower degree interpolations. No
	heap allocations will be performed.
- Fast: The code was written with auto-vectorization in mind and produces decent
	results (`clang -O2 -ffast-math` or `gcc -O3 -ffast-math`). MSVC
	unfortunately struggles a lot to produce decent assembly (likely due to
	heavy inlining).
- Type generic: The library functions are mostly intended for usage with
	`float`, `double` and `long double`, but any type that supports copying,
	addition, multiplication, division and conversion from `int` and `long long`
	is suitable.
- `constexpr` compatible: If the type argument supports `constexpr`, so will the
	functions in this library. If not, the functions remain usable for runtime
	evaluation.
- Almost no hard-coded limits: The degree of smoothness is capped at 10 which is
	far above anything anybody could reasonably desire. The number of dimensions
	is truly unlimited. Template instantiation depth and constexpr evaluation
	step limit will be the limiting factors.

Limitations of the library are:
- Heavily templated: consequences can be code bloat and high compilation times.
	As a consequence, this library is best used by including it in a code file
	that provides functions more closely tailored to your specific use case.
- Requires reference points to be arranged in an integer grid. Arbitrary
	reference points require a more sophisticated spline-interpolation algorithm.
- Poor performance for non-trivial types: The implementation assumes copying and
	destruction to be cheap operations (at least compared to addition and
	multiplication). Performance for types like `mpf_class` from GMP or types from
	the `boost::multiprecision` library that allocate heap memory is likely to
	suffer.
- Unstable optimization: Level and quality of optimization and thus the ultimate
	runtime performance can heavily rely on a multitude of factors. Among other
	things: compiler vendor and version, compiler settings, runtime platform ...
	Optimization will become more unreliable the larger the parameters (number of
	dimensions, degrees of smoothness, size of type parameter).
- NOT SUITABLE FOR SCIENTIFIC OR PRECISION ENGINEERING PURPOSES: The
	implementation was written to be one thing first and foremost: short. That
	means in turn that other things like minimization of rounding errors were
	not deemed of utmost importance. In fact, it is recommended to compile the
	code with switches like `-ffast-math` enabled to get that extra boost in
	performance, since the reordering of operations done by the compiler is
	unlikely to significantly worsen the quality of the results.

How does it work?
-----------------

The short version is that for an interpolation of `N`-th Degree, there is a
`2*N` by `2*N` matrix `M(N)` that when multiplied with a vector containing the
neighborhood on the left and a vector containing the powers of `x` until the
exponent `2*N-1` on the right will yield the interpolated value at position `x`:
```
result = {a(1-N), a(2-N) ..., a(N-1), a(N)} * M(N) * {1, x, x^2, ..., x^(2*N-1)}^T
```
Interpolation in multiple dimensions is equivalent to interpolation in one
dimension after the other. So the bulk of the effort lies in pre-computing the
matrices that we need to implement the algorithm. For this library this has been
done up to the 10th degree. Higher degrees are theoretically possible, but
exceed the limits imposed by our usage of `long long` as numerator and
denominator of the matrix elements. It is also extremely unlikely that you will
ever even need or want a degree greater than 4.

The runtime complexity lies somewhere around:
```
O(S1 * S2 * ... * SN + S1*S1 + S2*S2 + ... + SN*SN)
```
The memory complexity is roughly:
```
O(S1 + S2 + ... + SN + max(S1*S1, S2*S2, ..., SN*SN))
```

Q & A
-----

- Why would anybody use this?

	Ultimately this is a utility that can complement noise functions or any sort
	of creation process where a smooth interpolation is desired. It is not very
	well optimized and functions more as a proof of concept. However, if you
	ever have some sort of data consisting of grid-aligned reference points
	(e. g. a heightmap in an open world game) and wish a smooth interpolation
	that does only depend on local information, this library can be a useful
	utility at least until you get to the stage of low-level optimization in
	your project. You can also take the matrix data and implement similar
	interpolation algorithms in GPU shaders.

- What's up with the identifiers?

	All identifiers that are not part of the library's interface start with an
	underscore followed by a lowercase letter. According to the standard, all
	such identifiers are reserved at file scope, but free to use on all inner
	scopes (including within namespaces). This puts them at the unique postion
	of being the only identifiers that are *guaranteed* by the standard to not
	be macros. Thus I chose to implement the library using only such
	identifiers. If there are any name collisions whatsoever, the error likely
	lies outside of this header file.

	If you're a language lawyer, insisting that it's a terrible idea, feel free
	to write a lengthy blogpost about it.

- Am I allowed to use this in my project?

	Yes. As indicated in the header itself, this library is published under
  the Unlicense, meaning that you're free to use it for whatever purpose you
  seem fit. Under no circumstances are you obliged to seek my approval, give
  credit or share profits made from the usage of this library.
