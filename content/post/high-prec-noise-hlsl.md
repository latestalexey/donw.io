+++
date = "2016-08-25T22:05:56+01:00"
draft = false
title = "Mixed Precision GPU Noise with HLSL"
tags = [ "noise", "gpu" ]
+++

I posted an article a while back, entitled [Very Fast, High-Precision SIMD/GPU Gradient Noise]({{< relref	"high-prec-gradient-noise.md" >}}), where I outlined a technique for achieving double-resolution noise at speeds close to that when using float arithmetic. The key observation was that `floor` could be used on cell boundaries to mask off the ranges that require double arithmetic, allowing the bulk of the work to use float arithmetic.

<!--more-->

The GPU code was initially written in OpenCL and then ported to CUDA using [ComputeBridge](https://github.com/Celtoys/ComputeBridge). Neither were good platforms for releasing a game; releasing them on both at the same time was a recipe for madness so I ported everything to HLSL. Unfortunately HLSL SM5 doesn't support `floor(double)`.

So I sat down and took some time to cook up a software version. The first task was to isolate the fraction:

~~~cpp
double MaskOutFraction(double v)
{
	// Alias double as 2 32-bit integers
	uint d0, d1;
	asuint(v, d0, d1);

	// 0  ... 51   mantissa		0  ... 19
	// 52 ... 62   exponent		20 ... 30
	// 63 ... 63   sign

	// Already a fraction?
	int exponent = ((d1 >> 20) & 0x7FF) - 1023;
	if (exponent < 0)
		return 0;

	// Calculate how many bits to shift to remove the fraction
	// As there is no check here for mask_bits <= 0, if the input double is large enough
	// such that it can't have any fractional representation, thie function will return
	// an incorrect result.
	// As this is the GPU, I've decided against that branch.
	int mask_bits = max(52 - exponent, 0);

	// Calculate low 31-bits of the inverted mantissa mask
	uint lo_shift_bits = min(mask_bits, 31);
	uint lo_mask = (1 << lo_shift_bits) - 1;

	// Can't do (1<<32)-1 with 32-bit integer so OR in the final bit if need be
	lo_mask |= mask_bits > 31 ? 0x80000000 : 0;

	// Calculate high 20 bits of the inverted mantissa mask
	uint hi_shift_bits = max(mask_bits - 32, 0);
	uint hi_mask = (1 << hi_shift_bits) - 1;

	// Mask out the fractional bits and recombine as a double
	d0 &= ~lo_mask;
	d1 &= ~hi_mask;
	v = asdouble(d0, d1);

	return v;
}
~~~

With that you can then subtract the fraction and provide necessary overloads:

~~~cpp
// HLSL(SM5) doesn't support floor(double) so implement it in software
double Floor(double v)
{
	double r = MaskOutFraction(v);
	return v - r < 0 ? r - 1 : r;
}
double2 Floor(double2 v)
{
	v.x = Floor(v.x);
	v.y = Floor(v.y);
	return v;
}
double3 Floor(double3 v)
{
	v.x = Floor(v.x);
	v.y = Floor(v.y);
	v.z = Floor(v.z);
	return v;
}
double4 Floor(double4 v)
{
	v.x = Floor(v.x);
	v.y = Floor(v.y);
	v.z = Floor(v.z);
	v.w = Floor(v.w);
	return v;
}
~~~

Performance is admirably close to the CUDA/OpenCL versions (on nVidia/AMD hardware, respectively) and the same mask function can be reused for `Round` or `Ceil` functions.