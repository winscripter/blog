---
layout: post
title: "Fast H.264 Intra Prediction in .NET"
date: 2026-02-24
categories: blog
---

Today, let's push .NET to its limits. We are going to write a simple H.264 intra predictor in C# using a few libraries and SIMD.

### What is Intra prediction?
In H.264, Intra prediction refers to frames with I slices. Frames with P/B slices have to make use of past and future frames, which compresses much better since you reuse some info across frames than representing it over and over again. Intra frames however don't rely on past and future frames - they're a still image, almost like a JPEG file.

What they do is, across each macroblock, they make use of neighboring pixels. Those are pixels to the left, top, top right and top left of the active macroblock. Using those pixels, they apply one of common patterns to the macroblock.

Now, of course, these 9 patterns won't actually represent the entire macroblock. The idea is to help reduce residual coefficients. You apply patterns and then encode residuals to complete the frame.

### Dependencies
Let's start by installing `CommunityToolkit.HighPerformance`. Why? Because it gives us a `Span2D` struct which represents a 2D array which can be allocated on a stack.

### SIMD intrinsics
SIMD (Single Instruction Multiple Data) can be very helpful here because these are a single CPU instruction which can modify multiple values at once.

So if you wanted to add 4 integers at the same time, instead of a for loop, you can use SIMD which can be very quick.

.NET 5 comes with the `System.Runtime.Intrinsics` namespace to help with this. The core ideas are:
- Vectors: A CPU vector is a fixed-size "array" to store multiple values for use in SIMD. For instance, `Vector128` is just 128 bits which can be interpreted as 4 `ints`, 2 `doubles`, 8 `shorts`, however you'd like.
- Intrinsics: Actual access to SIMD intructions.

The JIT compiler automatically replaces calls to SIMD instructions with the SIMD instructions themselves.

For CPUs running x86/x64, you'd use SSE2 to process 128-bit vectors, AVX/AVX2 to process 256-bit vectors, or AVX512 to process 512-bit vectors.

> [!IMPORTANT]
> AVX512 is very rare. Only extremely high-end CPUs support it. I'm not talking about high-end gaming CPUs... I'm talking about those server CPUs, which cost from several thousands to tens of thousands of dollars in the U.S. AMD's top-notch CPU architecture, Zen5, also supports AVX512, though only the 9950X processor which is also very expensive.
>
> You're almost certain a home user's CPU won't have this.

For ARM64, you can use NEON (a.k.a. AdvSimd), and for WASM, you can use PackedSimd, both of which support 128-bit vectors only.

# Tables
The tables are in this form.

TL: Top left

TR: Top Right

T: Top

L: Left

P: Actual pixel buffer to predict.

| TL   | T0 | T1 | T2 | T3 |
| --   | ---- | ---- | ---- | ---- |
| L0 | P0,0 | P1,0 | P2,0 | P3,0 |
| L1 | P0,1 | P1,1 | P2,1 | P3,1 |
| L2 | P0,2 | P1,2 | P2,2 | P3,2 |
| L3 | P0,3 | P1,3 | P2,3 | P3,3 |

# From the Spec: the p[x, y] value
The p[x, y] value is a simple way to query left, top left, right, or top right pixels.

It is simply the following pseudocode.
```c
int* p(int x, int y) {
  if (x == -1 && y == -1) return &TL;
  else if (y == -1) return &T[x];
  else if (x == -1) return &L[y];
  else return &PixelBuffer[x, y];
}

// p(x, y) = value
void set_p(int x, int y, int value) { *(p(x, y)) = value; }

// p(x, y)
int get_p(int x, int y) { return *p(x, y); }
```

### Vertical/Horizontal
Vertical and Horizontal prediction in H.264 Intra is available for all macroblock sizes and YUV planes.

To keep this simple, let's just go over Vertical/Horizontal for now.

Example pattern for Vertical:
| 51   | 33 | 25 | 19 | 74 |
| --   | ---- | ---- | ---- | ---- |
| 16 | 33 | 25 | 19 | 74 |
| 29 | 33 | 25 | 19 | 74 |
| 42 | 33 | 25 | 19 | 74 |
| 57 | 33 | 25 | 19 | 74 |

As you can see, top pixels just get duplicated all the way down.

For Horizontal, it's the same, but instead of Top, you do Left now.

| 51   | 33 | 25 | 19 | 74 |
| --   | ---- | ---- | ---- | ---- |
| 16 | 16 | 16 | 16 | 16 |
| 29 | 29 | 29 | 29 | 29 |
| 42 | 42 | 42 | 42 | 42 |
| 57 | 57 | 57 | 57 | 57 |

Let's sketch out a class for Intra prediction.
```cs
using CommunityToolkit.HighPerformance;

// This is where T, TL, TR and L is located.
public readonly ref struct H264IntraBuffers
{
    public readonly Span<int> Top;
    public readonly Span<int> Left;
    public readonly Span<int> TopRight;
    public readonly int TopLeft;
}

public unsafe ref struct H264IntraPredictor
{
    public readonly H264IntraBuffers Samples;

    // For extremely fast access
    private readonly int* pTopRef;
    private readonly int* pLeftRef;

    public H264IntraPredictor(H264IntraBuffers buffers)
    {
        Samples = buffers;

        this.pTopRef = (int*)Unsafe.AsPointer<int>(ref buffers.Top.DangerousGetReference());
        this.pLeftRef = (int*)Unsafe.AsPointer<int>(ref buffers.Left.DangerousGetReference());
    }
}
```

Let's write a simple implementation.

```cs
public unsafe void Vertical(Span2D<int> output)
{
    for (int x = 0; x < output.Width; x++)
    {
        for (int y = 0; y < output.Height; y++)
        {
            output[x, y] = this.pTopRef[x];
        }
    }
}

public unsafe void Horizontal(Span2D<int> output)
{
    for (int x = 0; x < output.Width; x++)
    {
        for (int y = 0; y < output.Height; y++)
        {
            output[x, y] = this.pLeftRef[y];
        }
    }
}
```

# What we learned
We learned how to use CommunityToolkit.HighPerformance, unsafe code, and mainly, Vertical/Horizontal H.264 predictions.

Next up, I will actually show you how we can optimize Vertical/Horizontal methods to use SIMD intrinsics.
