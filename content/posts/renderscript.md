+++
author = "pshegger"
title = "RenderScript is Still alive"
date = "2020-02-29"
description = "RenderScript is a great way to compute large amount of data, so let's discover together what it is and when should it be considered."
tags = [
    "programming",
    "android", 
    "renderscript",
    "parallelism"
]
categories = [
    "android"
]
toc = true
related = true
social_share = true
+++

I'm sure for most of you it's not a surprise, but I also think that there are people out there who either doesn't know what it is or simply tends to forget about it. The aim of this article is to show these people what it is and why should they use it.

## So, what *is* RenderScript?

RenderScript is an Android framework which can be used for running computationally heavy tasks at high performance. You will get the most benefit on using it for tasks which require a lot of computation that can be parallelized, as the framework will run your code on every available CPU and GPU cores.

The first step when writing a task for RenderScript is writing the script file. This file will contain your kernel(s) and everything else needed to compute your result. The kernel is the function which will be executed on your input data in parallel. There are two type of kernels: *mapping* and *reduction*, but in this article we will only discuss the former one.

## Testing method

As I mentioned before RenderScript is excellent at performing computationally heavy tasks and to demonstrate that I created a simple test application. For this test we will be generating an image of the [Mandelbrot set](https://en.wikipedia.org/wiki/Mandelbrot_set) using different methods. Our first version will be written in Kotlin and will not contain any optimization. The second version will still be in Kotlin but we will try to use it's built-in features to parallelize the computation. Lastly we will have a version written in RenderScript.

## Mandelbrot

Generating an image of the Mandelbrot set is fairly easy, but it requires a lot of time. All we have to do is calculate for every pixel whether the function `f(z) = z*z + c` (where `z=0+0i` and `c` is the coordinate of the pixel mapped to the complex plane) goes to infinity or not.

As there is no simple way to decide that in practice we usually run a predetermined number of iterations and see if the result leaves a predetermined neighborhood of 0. This means that we can make more precise estimates with choosing a higher iteration count, but it will also slow down the process.

After that we just have to take the number of iterations it took to leave the neighborhood and color the pixel accordingly.

## Kotlin - the easy one

With this knowledge we can easily write a method which calculates the color for a given coordinate.

```kotlin
protected fun calculatePoint(x: Int, y: Int, rect: RectF): Int {
    val cr = rect.left + (rect.right - rect.left) * (x / width.toDouble())
    val ci = rect.top + (rect.bottom - rect.top) * (y / height.toDouble())
    var zr = 0.0
    var zi = 0.0
    
    var i = 0
    while (i < MainActivity.MAX_ITER && zr * zr + zi * zi < 4) {
        val nzr = zr * zr - zi * zi + cr
        val nzi = 2 * zr * zi + ci
        zr = nzr
        zi = nzi
        i++
    }
    
    val hue = 360 * i.toFloat() / MainActivity.MAX_ITER
    val sat = 1f
    val value = if (i < MainActivity.MAX_ITER) 1f else 0f
    return Color.HSVToColor(floatArrayOf(hue, sat, value))
}
```

This function can be called with the coordinate of the pixel we want to calculate (`x` and `y`) and the specific area of the plane we want to display currently (`rect`). Using these inputs we can then calculate the coordinate on the plane (`cr`and `ci`). We could also use some complex number class, but from my experience that leads to a performance drop so we stick to calculating the real and imaginary part separately. After that we just have to loop until we either reach the maximum iteration count or leave the neighborhood of 0.

The last step is coloring the pixel. For that the easiest way to do is using the HSV color model as you can see in the code above.

Now that we have everything to calculate the color of one pixel all we have to do is looping over every pixel of the output image and coloring it.

## Kotlin Parallel - the complicated one

For this version will use the same function to calculate the color of one pixel, but we will split the output to multiple regions and calculate them in parallel. The number of partitions must equal to the number of threads we are using for calculation, but other than that we can split any way we would like to.

To determine the number of threads we will use `Runtime.getRuntime().availableProcessors() - 1`. The `-1` ensures that we have one core for our main thread. When we have the number of threads we create a `CoroutineDispatcher` and launch a new scope using that as it's context.

At this point we have to loop through our partitions and use `CoroutineScope.launch` to start the calculation on it's own thread. This calculation is the same as before, but we're just calculating the points for the given region and not for the whole image.

```kotlin
override suspend fun generate(buffer: Bitmap, rect: RectF) {
    val numThreads = Runtime.getRuntime().availableProcessors() - 1
    val context = Executors.newFixedThreadPool(numThreads).asCoroutineDispatcher()
    
    var h = sqrt(numThreads.toDouble()).toInt()
    while (numThreads % h != 0) h--
    val w = numThreads / h
    
    val segWidth = width / w
    val segHeight = height / h
    
    val canvas = Canvas(buffer)
    
    val calculation = CoroutineScope(context).launch {
        for (mod in (0 until numThreads)) {
            val segX = mod % w
            val segY = mod / w
            val segXStart = segWidth * segX
            val segXEnd = segWidth * (segX + 1)
            val segYStart = segHeight * segY
            val segYEnd = segHeight * (segY + 1)
            val paint = Paint()
    
            launch {
                for (y in (segYStart until segYEnd)) {
                    for (x in (segXStart until segXEnd)) {
                        val result = calculatePoint(x, y, rect)
    
                        paint.color = result
                        canvas.drawPoint(x.toFloat(), y.toFloat(), paint)
                    }
                }
            }
        }
    }
    calculation.join()
}
```

## RenderScript - the fast one

As the previous example shows it can be pretty tricky to parallelize our computations. We have to manually create our threads and feed them the correct values. But this is where RenderScript comes to rescue us.

RenderScript will automatically manage the parallelization, all we have to do is write the function which will be executed for every input data (this will be our `kernel`). The language is a derivative of C99 so some familiarity with that is required to use it.

For our example all we have to write is a simple script which contains every input we will need for the calculation (width/height of the output image, the area of the plane we want to calculate and the number of iterations) and our kernel, which will be basically the same we have written in Kotlin to calculate the color of a single point. (We will also have to write a color converter, but that's only specific for this example.)

```c
#pragma version(1)
#pragma rs_fp_full
#pragma rs java_package_name(io.github.pshegger.playground.rsperformance)
    
int width, height;
float left, right, top, bottom;
int maxIterations = 128;
    
static uchar4 getColor(int i) {
    float hue = 360 * (float) i / maxIterations;
    float sat = 1;
    float val = (i < maxIterations) ? 1 : 0;
    float c = val * sat;
    float k = hue / 60;
    float kmod = fmod(k, 2);
    float kmodAbs = (kmod - 1 < 0) ? -(kmod - 1) : (kmod - 1);
    float x = c * (1 - kmodAbs);
    float r1 = 0, g1 = 0, b1 = 0;
    
    if(k>=0 && k<=1) { r1=c; g1=x; }
    if(k>1 && k<=2)  { r1=x; g1=c; }
    if(k>2 && k<=3)  { g1=c; b1=x; }
    if(k>3 && k<=4)  { g1=x; b1=c; }
    if(k>4 && k<=5)  { r1=x; b1=c; }
    if(k>5 && k<=6)  { r1=c; b1=x; }
    
    float m = val - c;
    return rsPackColorTo8888(r1 + m, g1 + m, b1 + m);
}
    
uchar4 RS_KERNEL mandelbrot(uchar4 in, uint32_t x, uint32_t y) {
    float nzr, nzi;
    float zr = 0, zi = 0;
    float cr = left + (right - left) * (x / (double) width);
    float ci = top + (bottom - top) * (y / (double) height);
    
    int i = 0;
    while (i < maxIterations && zr * zr + zi * zi < 4) {
        nzr = zr * zr - zi * zi + cr;
        nzi = 2 * zr * zi + ci;
        zr = nzr;
        zi = nzi;
        i++;
    }
    
    return getColor(i);
}
```

As you can see the kernel (marked with `RS_KERNEL`) is basically the same as was before with just some minor changes required by the language. It is also worth noting that the kernel's `x` and `y` parameter will not be provided by us, but the framework.

We have to put this file inside it's own directory located at `app/src/main/rs` and we will call it `mandelbrot.rs`. After saving we can build the project so the required java files will be generated and after it's finished we can start using it in our application.

There are a few setup steps before we can really use it. First we have to create a RenderScript context and use that to initialize our script. We also have to set the global variables for the script and create allocations for our input and output.

After all these steps we are ready to start the calculations by calling our kernel on the loaded script.

```kotlin
private val rs = RenderScript.create(context)
private val script = ScriptC_mandelbrot(rs).apply {
    _width = width
    _height = height
    _maxIterations = MAX_ITER
}
    
override suspend fun generate(buffer: Bitmap, rect: RectF) {
    script.apply {
        _left = rect.left
        _top = rect.top
        _right = rect.right
        _bottom = rect.bottom
    }
    
    val bufferAllocation = Allocation.createFromBitmap(rs, buffer)
    script.forEach_mandelbrot(bufferAllocation, bufferAllocation)
    bufferAllocation.copyTo(buffer)
    bufferAllocation.destroy()
}
```

As there can be more than one kernels in the script we have to specify which one we would like to use by calling `forEach_{{kernelName}}`. As we are not using our input for anything we can also save some memory by using that for our output too.

After all that we only have to copy our allocation back to a bitmap and we can display that to show our users the result.

## So, was it worth it?

Now that we're ready with our 3 implementations we can compare them.

We can easily see that the easiest one to write was the Kotlin version. We had to do some extra steps to write the RenderScript version, but it was still less hassle than creating our own parallel implementation.

And what about performance? To measure that I ran every computation a few times and measured the average duration it took to generate a single image. The order of the methods is as expected, Kotlin is the slowest with an average of `1016 ms/frame`, the second one is the parallel version with an average of `475 ms/frame` and the winner is RenderScript with an average of `33 ms/frame`. Of course the numbers my vary, but we can easily see that RenderScript is the fastest by a big margin.

In my opinion the extra work easily worths the performance gain, but of course everybody should decide it for themselves.

## When should I use it?

We have seen that it can be used with most efficiency when parallel computing can drastically increase the performance of our code. This is usually true for those scenarios when a large number of things should be computed and they are independent of each other.

RenderScript (as the name suggests) is most used for image processing/manipulation, but I'm sure that there are a lot of other areas of computing where it could be used effectively.

And just to have a real life example: I'm sure a lot of us had met a designer in the past who wanted some fancy blur effect for some dynamic parts of the UI which is almost impossible to achieve normally. From now on we don't have to say that we can't do that we just have to tell the truth:

> It is possible, but will take longer.

## Further reading

You can read more about RenderScript on the [Android Developer Site](https://developer.android.com/guide/topics/renderscript/compute), and check out the source code of the test application at [https://github.com/PsHegger/render-script-performance](https://github.com/PsHegger/render-script-performance)