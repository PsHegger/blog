+++
author = "Gergely Hegedüs"
title = "Help Yourself and the Compiler with Contracts"
date = "2019-03-21"
description = "In the world of Kotlin, contracts represent a deal between the developer and the compiler. As a developer you can share insight of your code with the compiler and it can use this extra info for better code analysis."
tags = [
    "programming",
    "kotlin",
    "contracts"
]
categories = [
    "kotlin"
]
+++

## What are contracts?

In the world of Kotlin, contracts represent a deal between the developer and the compiler. As a developer you can share insight of your code with the compiler and it can use this extra info for better code analysis.

Currently there are two kinds of contracts available: `callsInPlace` which tells the compiler how many times a lambda is called, and `returns`/`implies` which indicates that a defined condition is true if the function returns a specific value.

## Basic usage

In order to write contracts for your own functions you have to keep in mind a few things. First of all this feature is experimental in Kotlin 1.3, so you have to annotate your functions with `@ExperimentalContracts`. These functions also have to be top-level functions or your code will fail to compile.

With these in mind let's see how a function with a contract looks like:

```kotlin
@ExperimentalContracts
fun foo() {
    contract {
        // your contract definition
    }
    // some other code
}
```

To write the actual definition we will be using the Contract DSL.

## callsInPlace

Let's take a look at the following example: you want to put a delay in your code, but the value and the unit is stored separately. You decide to write a function which reads the values and passes them back using a lambda.

```kotlin
fun readDelay(callback: (Long, TimeUnit) -> Unit) {
    val value = readDelayValue()
    val unit = readDelayUnit()
    callback(value, unit)
}
```

You would like to store the values for later use so you create variables for them:

```kotlin
val delayValue: Long
val delayUnit: TimeUnit
    
readDelay { value, unit ->
    delayValue = value
    delayUnit = unit
}
    
Thread.sleep(delayUnit.toMillis(delayValue))
```

The problem is that this code will not compile because the compiler has no idea how many times the lambda will be called. If it's not called at all the two variables would be uninitialized and if it's called more than once they would be reassigned, which is forbidden. Luckily we have `callsInPlace` which is exactly what we need now. Let's modify our reader function:

```kotlin
@ExperimentalContracts
fun readDelay(callback: (Long, TimeUnit) -> Unit) {
    contract {
        callsInPlace(callback, InvocationKind.EXACTLY_ONCE)
    }
    
    val value = readDelayValue()
    val unit = readDelayUnit()
    callback(value, unit)
}
```

The main logic is the same as before, but we added a contract to our function. This tells the compiler that the lambda will be called exactly once (`InvocationKind.EXACTLY_ONCE`). With that knowledge the compiler will know that the variables will be initialized and will not be reassigned.

There are 3 other possible invocation kinds: `AT_MOST_ONCE` (the lambda will be called once or not at all), `AT_LEAST_ONCE` (it will be called one or more times) and `UNKNOWN` (it will be called, but we don't know how many times).

## returns/implies

The other type of contract we can write tells the compiler that a specific condition is true if the function returns with the specified value. Let's take a look at the following code:

```kotlin
@ExperimentalContracts
fun String?.isValid(): Boolean {
    contract {
        returns(true) implies (this@isValid is String)
    }
    
    return this != null && this.length > 3
}
```

We can see that other kind of contract in this example, but what does it exactly do? To understand it better let's break it down to pieces. The first part (`returns(true)`) indicates that the contract applies when this function returns `true` as its result, and the second part tells the compiler what to expect from it. In this case it indicates that the particular `String?` instance we're calling this function on is in fact a `String` and in reality it's not nullable. From this the compiler knows that if the function returns `true` it's safe to smart-case the instance to a non-nullable type. This is how it looks when we try to use it:

```kotlin
val result: String? = getResult()
        
if (result.isValid()) {
    println("The length of the result is ${result.length}")
}
```

As you can see we didn't have to use `String?.length`, because `result` has been smart-casted to `String`. Without the contract we would either had to check for nullability in the `if` expression or use the safe call operator (`?.`).

Currently there are 3 types of return state indicators: `returns` (without any argument) expresses that the function returned successfully, `returns(Any?)` (currently only `true`, `false` and `null` is supported) expresses that the function returned successfully with the given value and `returnsNotNull` which expresses that the returned value is not null. The syntax is the same for all 3 types: `*type* implies *expression*`.

## A few things to keep in mind

When writing your contracts there are a few things to keep in mind. First of all, contracts are not verified, which means that you can write a function with a wrong contract and this may lead to unexpected errors at **runtime**.

The other important thing to remember is that contracts are an experimental feature in Kotlin 1.3 and because of that the syntax may change in the future, so you might want to wait until its stable release to use it in production.