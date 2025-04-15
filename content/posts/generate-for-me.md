---
author: "pshegger"
title: "@GenerateForMe"
date: "2019-11-11"
description: "Annotations are extremely versatile especially if we write our own processor to overcharge our development. Learn how to write one in Kotlin."
tags: ["programming", "kotlin", "annotation", "kapt"]
categories: ["kotlin"]
---

## Write once, use everywhere

Sooner or later every developer meets the following scenario: you have written the same piece of code multiple times, maybe with some minor differences, and you don't want to do it again. You analyze the situation and decide to create a separate function so the next time you need it you just have to call it. Sometimes it's not enough so you create a new abstraction layer and write a class which does what you need. But what happens if you cannot create an abstraction which is suitable for you *and* it's easy to use whenever you need it. That's when Annotation Processing comes for your help.

When you write an annotation processor you are creating an application which writes complex code instead of you, but based on your rules. This of course isn't the solution every time as writing these processors could easily end up taking more time than you can save with it ([relevant XKCD](https://xkcd.com/1205/)), but sometimes it can really help your productivity.

## The example

Let's write an annotation processor which creates our `RecyclerView` adapter and view holder. If you're not an Android developer don't be alarmed, the same technics can be used for any type of Kotlin code.

Writing the adapter is usually pretty straightforward, but you have to do it for every model you want to display inside a `RecyclerView`, and writing it generates a lot of boilerplate. Naturally we can't eliminate every parts of writing it, but we can make it easier and faster.

## How to start?

Before we do any coding we should plan how our interface will look like for the end user. As we are planning to generate the adapter with the least possible code we should use a single class for the base and its functions will be used to define the bindings between the data and the view. This means that we will need 2 annotations: one for the class which will define the layout file and an other one for the binding functions, which will define the id of the specific view. Let's call them *@ModelBinding* and *@BindView*. Both of them will have one parameter, for *@ModelBinding* this will be the layout file for the View and for *@BindView* this will be the ID of the specific View. We will also have a few restriction on where the end user will be able to use these annotations. *@ModelBinding* will only be usable for classes which has a constructor with a single parameter (this will be the item we will bind to the View) and *@BindView* will only work for single parameter functions inside the class. Let's see an example:

```kotlin
@ModelBinding(R.layout.item_user)
class UserBinding(private val user: User) {
    
    @BindView(R.id.name)
    fun bindName(name: TextView) {
        name.text = user.name
    }
    
    @BindView(R.id.email)
    fun bindEmail(email: TextView) {
        email.text = user.email
    }
}
```

This is all the code we will have to write to create an adapter for our User model which displays the user's name and email address.

## Project setup

As we are planning to enchant our existing code with the generator we will be starting by opening the project. If you want to create the generator as a library the steps are almost the same, except we do not have to add our new modules as a dependency.

First we have to create a new (Java library) module for our annotations. This module will only contain our annotation classes, so no dependency is needed (beside Kotlin). After that let's add our classes.

```kotlin
// BindView.kt
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.SOURCE)
annotation class BindView(val viewId: Int)

// ModelBinding.kt
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class ModelBinding(val layoutId: Int)
```

The next step is creating our generator module. This also has to be a Java library, but it will have significantly more code in it. From this point on we will only be working in this module.

First we have to add a few dependencies:

```gradle
implementation(project(":annotations"))
implementation("com.google.auto.service:auto-service:1.0-rc4")
implementation("com.squareup:kotlinpoet:1.3.0")
    
kapt("com.google.auto.service:auto-service:1.0-rc4")
```

We have to add our previous module as well as Google's AutoService and KotlinPoet.

AutoService is a useful tool which generates configuration for `ServiceLoader` so we don't have to set up our annotation processor by hand.

## KotlinPoet

We will be using [KotlinPoet](https://github.com/square/kotlinpoet) by Square to make our lives easier. It is not required but the alternative would be to write the whole generated code by hand. Let's take a look at some of its features.

### FileSpec

Using KotlinPoet's `FileSpec` we get a few advantages. First of all it handles saving the file, we just have to pass an instance of `java.io.File` to its `writeTo` method. The second and probably more important feature is that it automatically adds the imports so we don't have to bother about them.

```kotlin
val fileBuilder = FileSpec.Builder("package", "fileName")
// add whatever you need to the file
fileBuilder.build().writeTo(outputFile)
```

You can add everything which is valid in a Kotlin file, but the most important for us now is `addType`. In our case it is used to add our generated classes, but in other cases it can be used with any TypeSpec.

### Building a class

As mentioned above we will be using `TypeSpec` for generating our class. I find learning by examples the easiest so let's see some basic code and understand together what it does.

```kotlin
TypeSpec.classBuilder("Greeter")
    .primaryConstructor(
        FunSpec.constructorBuilder().addParameter("name", STRING).build()
    )
    .addProperty(
        PropertySpec.builder("name", STRING)
            .initializer("name")
            .addModifiers(KModifier.PRIVATE)
            .build()
    )
    .addFunction(
        FunSpec.builder("greet")
            .addStatement("println(\"Hello $name\")")
            .build()
    )
    .build()
```

This code will generate the following class:

```kotlin
class Greeter(private val name: String) {
    fun greet() {
        println("Hello $name")
    }
}
```

So, what's happening here. In my opinion KotlinPoet's API is pretty straightforward and reading it makes everything clear but let's work ourselves through the code together.

First of all we are defining a class called `Greeter` using `TypeSpec.classBuilder`. After that we are adding 3 things to the class: a constructor, a property and a function. For this we are using `FunSpec` for functions (for the constructor we have to specify that it is in fact not a normal function, for this we are using `constructorBuilder`), and `PropertySpec`. After everything is added we just call build and we have our class as a `TypeSpec`. We can then add it to our `FileSpec` and save it.

## Generating the ViewHolder

Most of the code in this step is easy after learning the basics. We need to create a class which subclasses the `RecyclerView.ViewHolder` abstract class. It has to have a constructor which accepts a `View` as a parameter to pass it to the superclass. If we have that all that remains to write is the main part of the `ViewHolder` which binds the data to the view. We will call this function `bind`.

To generate it we have to iterate over the functions marked with `@BindView` in the binding class. We can do that using `Element.getEnclosedElements()` which returns all elements inside an other one, and we can filter for the ones that are methods and has our annotation. After that we have to add each `View` as a class member for the `ViewHolder` and will be initialized with the ID which was passed to the annotation as a parameter. We will also have to call every function inside bind.

```kotlin
fun generateViewHolder(element: Element, itemPack: String, itemClassName: String): TypeSpec {
    val classBuilder = TypeSpec.classBuilder("${itemClassName}ViewHolder")
        .primaryConstructor(
            FunSpec.constructorBuilder()
                .addParameter("itemView", ClassName("android.view", "View"))
                .build()
        )
        .superclass(ClassName("androidx.recyclerview.widget.RecyclerView", "ViewHolder"))
        .addSuperclassConstructorParameter("itemView")

    val binderName = getClassName(element)

    val bindFunBuilder = FunSpec.builder("bind")
        .addParameter("item", ClassName(itemPack, itemClassName))
        .addStatement("val binder = ${binderName.canonicalName}(item)")

    element.enclosedElements
        .filter { element ->
            element.getAnnotation(Bind::class.java) != null &&
                            element.kind == ElementKind.METHOD &&
                            (element as ExecutableElement).parameters.size == 1
        }
        .forEach { element ->
            val bind = element.getAnnotation(Bind::class.java)
            val p = (element as ExecutableElement).parameters[0]
            val viewName = p.simpleName.toString()

            classBuilder.addProperty(
                PropertySpec.builder(
                    viewName,
                    getClassName(p),
                    KModifier.PRIVATE
                )
                    .initializer("itemView.findViewById(%L)", bind.viewId)
                    .build()
            )

            bindFunBuilder.addStatement("binder.${element.simpleName}($viewName)")
    }

    return classBuilder.addFunction(bindFunBuilder.build()).build()
}
```

If we run it with the previous binding example it will generate the following code (after some formatting):

```kotlin
class UserViewHolder(itemView: View) : ViewHolder(itemView) {
    private val name: TextView = itemView.findViewById(11111)
    private val email: TextView = itemView.findViewById(22222)
    
    fun bind(item: User) {
        val binder = UserBinding(item)
        binder.bindName(name)
        binder.bindEmail(email)
    }
}
```

The view IDs are just examples. When passing them to the annotation we lose the name and can only use the real value, but it's not a problem, because they represent the same thing.

## Generating the Adapter

We are done with the hard part but we still need to generate the adapter. For this we need to generate a class which overrides the methods of `RecyclerView.Adapter` and for the real binding we will use our `ViewHolder`. We already know almost everything to write the code which will generate it, but there is one thing we didn't discuss yet. We will need to use generic classes which can be achieved by calling `parameterizedBy` on a `ClassName` instance. So for example in our case we can create the list of items with the following code:

```kotlin
ClassName("kotlin.collections", "List")
    .parameterizedBy(ClassName(itemPack, itemClassName))
```

We can then pass it as any other type to KotlinPoet and in the generated code it will show up as `List<User>`.

## Tying it all together

The final step in the puzzle is to create our processor. This is the part of the code generator which will be called by kapt and is responsible for handling the annotations.

Writing the skeleton of the class is self-explanatory. It has to extend the `AbstractProcessor` class which has only one abstract method: `process`. For us the second parameter is the more important, which is a `RoundEnvironment`. With that we can get everything annotated with our `ModelBinding` annotation. After that we just have to loop through these elements which contains most of the info we will need to use our adapter builder.

We also need to use `processingEnv` which is a protected member of `AbstractProcessor`. This is the bridge between our processor and the code we are working with. We will use this for 3 things now: getting the folder where we should save our generated files, getting package info about the classes we are working with and printing diagnostic messages.

Kapt will save the directory for generated files into the processing environment's options with the name `kapt.kotlin.generated`. We will be also using a simple data class called `AdapterInfo` to pass every needed info to our code generator classes.

Now that we have every knowledge let's see how our final process method looks like:

```kotlin
override fun process(annotations: MutableSet<out TypeElement>?, roundEnv: RoundEnvironment): Boolean {
    val kaptKotlinGeneratedDir = processingEnv.options["kapt.kotlin.generated"] ?: return false
    
    roundEnv.getElementsAnnotatedWith(ModelBinding::class.java)
        .mapNotNull { element ->
            if (element.kind != ElementKind.CLASS) {
                processingEnv.messager.printMessage(
                    Diagnostic.Kind.ERROR,
                    "Only classes can be annotated with @ModelBinding"
                )
                return@mapNotNull null
            }
            generateAdapterInfo(element)
        }
        .map { adapterInfo ->
            val fileName = "${adapterInfo.itemClassName}Adapter"
            val fileBuilder = FileSpec.builder(adapterInfo.pack, fileName)
            fileBuilder.addType(AdapterBuilder(processingEnv, adapterInfo).build()).build()
        }
        .forEach { fileSpec ->
            fileSpec.writeTo(File(kaptKotlinGeneratedDir))
        }
    
    return true
}
```

## Using it

We are now finished with the processor and all that left is using it inside our main module. For that we have to add the two new modules as a dependency.

```gradle
implementation(project(":annotations"))
kapt(project(":codegen"))
```

Once they're added we are ready to use it. All we need is a layout for our recycler item and a `RecyclerView`. After that we just have to write our class containing our bindings and press build.

If we did everything correctly, once the build is finished there will be a newly generated adapter which we can then use with our `RecyclerView`s.

## Sample Code

If you would like to have a look at the final version of the code you can do that at [https://github.com/PsHegger/recycleradapter-generator/tree/0.1.0](https://github.com/PsHegger/recycleradapter-generator/tree/0.1.0).

It is also a bit extended and works as a standalone library, so if you're an Android developer and would like to use it in your project feel free to do so.

---
This post was originally published on [kotlindevelopment.com](https://www.kotlindevelopment.com/). If you liked it please consider visiting and reading some other articles there.
