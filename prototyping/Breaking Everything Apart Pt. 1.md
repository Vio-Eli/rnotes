# Introduction

![[Pasted image 20240806223737.png]]

## Python is Ass~~ymptotically Beautiful~~

We love python. Python is awesome. It's *easy-to-read*, *fast*, and most importantly, has error messages as beautiful as java (with the needle in the haystack search being the charm of them).

Okay, actually being truthful here, we hate python. Don't get me wrong, its a great beginner language. Key word: *beginner*. The second you try to actually build something, it becomes a clunky, convoluted mess -- why all of the main libraries used in it are written in some flavor of C.

So, why not write everything in C++? Well, let me introduce to you to my colleague who has spent all of 25 years coding C++, and still hasn't mastered it. It's *too* big, way too low level, and it's a pain to write in (there are few if no error messages at all, you just debug). With that, we need something that actually can do things and won't holistically rely on another language, but also provides coders with some level of support.

This is where Rust comes into play. It's a beautifully written language (except the Orphan rule, fuck the Orphan rule). It has extremely good error messages, and it's powerful enough to suit our purposes. But most important of all, it's learning curve isn't that bad.

But where do we go from here? We want to do Rust, but what will make us so special? 
*What about symbolic based mathematics?*
And thus, The RSeries was born. To recreate the beauty of numpy, gnu, matplotlib, etc., even tensorflow and pytorch, but in Rust -- with a special symbolic add-on.

## The Plot

So, The RSeries is huge, like ginormously huge (and will only get bigger with time). Imagine RSeries as this giant system with each RPart being their own planet with moons, each having it's own entire different ecosystem - different flora, fauna, aliens, even some cultural elements. But, each planet depends on the others for some things, maybe one planet has X resource but they need Y from another planet to survive... so they trade.

In the following babble I break these planets down to their very particles, down to the clumps of elements that make each and everything up, to the core of their logics. All Starting with RNum (red in the diagram above).

# RNum

## A Synopsys

The core mission sentiment of is to recreate numpy and scipy with symbolic based mathematics. This is achieved by using a variety of different techniques and methods that are exposed to the user.

## The Function

We can very easily express a function $y = x^2$ by simply writing it. That is, very obviously one of the first quadratic parabolas you ever learn. It's roots are $(0,0)$, and it's derivative is $y' = x$. When we want to find $y$ at say $x=3$, we simply square $x$ resulting in $y(x=3) = 9$. If we want to find the result at another value, we just square the value, and so on. This applies to pretty much any function.

Now, unfortunately, a computer doesn't quite work like a human (obviously, but bear with me). When we define some variable $x$ to be some value say $3$, all $x$ is essentially is a pointer to a memory block that holds a value of $3$. Nothing more, nothing less. So, when we have the function $y = x^2$, since $x$ isn't a number of some precision or a string (well, it could be a string, but it's pretty easy to see why that's horrible), we can't quite store it.

Thus, we use an enum. Essentially a function can be represented as a binary tree (I will simplify things a bit to make it easier to read):
$y = x^2$ is represented as `Pow(Var("x"), 2)`, or `Var("x").pow(2)`
We have most of the operations defined already.
- Var -> Variable
- Const -> Constant
- Add -> Addition
- Sub -> Subtraction
- Mul -> Multiplication
- Div -> Division
- Pow -> Power of
- etc... (`todo!()`)

Now, I want to explain some of the current logic behind the function enum, starting with the variable and constant types. See, when I noted earlier that it's bad to use strings, I sort-of lied. It's horrible to do operations on strings, however using them to store a "Variable", which is in reality just a letter, is actually a good use for them (or so we currently think). So, in essence, our `Var("variable letter")` is our "string" variable placeholder. The same idea goes for Constants, or `Const`, just instead of a string its an `f64`. It is compatible with anything that can be type-casted to `f64`.

Moving forward, most "operations" have defined fields. For example, $x - y$ which is written as `Sub(x, y)` (I omitted the Var()'s for simplicity) can't be written as `Sub(y, x)` because commutativity doesn't hold for subtraction. The same goes for division, and pretty much every operation, except Addition and Multiplication. In fact, in the first version of the Function enum, Addition and Multiplication had distinct lhs's and rhs's. However, that led to a very odd issue.

It's incredibly hard to *fast-simplify*. In the old version (where Add and Mul weren't vectors), Something like $x + 2 + y$ would look like `Add(x, Add(2, y))`. It's pretty obvious why this is problematic... something akin to $(x + y) + (x + 3)$ would be a nightmare to simplify. We humans can easily see it's $2x + y + 3$ but the algorithm to even get to that would need to cover *every* edge case, and there's a lot. In coding terms, $(x + y) + (x  + 3)$ is expressed as `Add(Add(x, y), Add(x, 3))`. To simplify that, we'd need to essentially flatten the Add tree and combine like terms, which is precisely why Add and Mul are both Vec's and don't have distinctive lhs and rhs's.

## The Simplify


### The Gist

We need some way of fast simplifying complex operations, because essentially every time we combine or do some operation, we want to limit the explosive growth of the Function tree. Essentially, all Simplify is is pruning that tree. However, we don't need a full *human* readable simplify -- we only need 80% simplify really. The goal here is to cut down on the number of operations needed to evaluate the tree at some given value(s) while not spending more time doing the simplify than the eval would take to begin with.

### Some Examples: (Scaling from simple to complex)

| Math Function   | Simplified Ver |
| --------------- | -------------- |
| $x + y + x$     | $2x + y$       |
| $2*(x + y) + x$ | $3x + 2y$      |
|                 |                |
|                 |                |
|                 |                |
|                 |                |
|                 |                |


### The algorithm

Our input is just a Function and our output needs to be another, albeit simplified, one.

Let us focus on just Add, Sub, Mul, Div, and Pow. Those 5 are essentially the "core". Pretty much everything revolves around those 5. Why is Pow included you ask? Because, we have a choice in how to express something like $x^2$. Is it `Mul[x, x]` or `Pow(x, 2)`. We chose to use `Pow(x, 2)` because of "how" we simplify a multiplication chain. When we have something like $10*x^2*y*z^3$ it's much easier for us to *contain* the variables with `Pow`. I mean, which would you chose? `Mul[10, x, x, y, z, z, z]` or `Mul[10, Pow(x, 2), y, Pow(z, 3)]`. From an algorithmic perspective, using `Pow`'s allows each variable to be "itemized" easier. And this leads me to the crux of the biggest of issues (so far) with the simplify.

How do we simplify something like $10xy + 2xy$, because that's actually written like `Add[Mul[10, x, y], Mul[2, x, y]]`, which, to us humans is extremely easy to be "oh both $xy$ are the same across each statement, so let us just add the constant's together." But, unfortunately, algorithms work slightly different. Because its not necessarily known exactly what you will be comparing, and the edge cases are essentially infinite, we need a way to "itemize" things.

Say we have some equation $A + B$, `Add[A, B]`. Since the entire function is a binary tree

