# Julia Advice for MATLAB (or R or Python) programmers

Here are a few things I've learned while transitioning from a mix of MATLAB and C to the [Julia](https://julialang.org) programming langage, starting around 2018. Some of this is rather straightforward, while other points may be less obvious:

#### At the REPL:
* Typing `?` followed by a function name will give you help/documentation on that function
* Typing `]` opens the package manager
* Typing `;` gives you the system command line
* Typing `@less` followed by a function call will show you the source code for that function

#### Common 'gotcha's:
(see also: https://docs.julialang.org/en/v1.5/manual/noteworthy-differences/)
* if `A` is an array, **assigning** `B = A` will copy`A` *by reference* such that both `A` and `B` point to the same memory,  i.e., if you subsequently change `A[1]`, that'll also change `B[1]`. If you don't want this, you need to make a copy, e.g. `B = copy(A)`.
* Slicing an array by **indexing**, with (e.g.)  `x = a[:, 5]` or `a = x[x .> 5]`, etc., *makes a copy*. This is great if you *want* a brand new array, but can be slow if you don't. In the latter case, you can instead use a [`view`](https://docs.julialang.org/en/latest/base/arrays/#Base.view), e.g. `view(a, :, 5)`, which will be much much faster where applicable. 
  * You can turn array-style indexing into a view with the `@views` macro (`@views a[:, 5]` equals `view(a, :, 5)`).
* For **element-wise** operations on an array, you put a dot on _every_ function/operator (not just a some, as in Matlab). 
  * This includes functions you call with `()`. For example, `f.(x)` is different than `f(x)`.
  * This known as **dot-broadcasting**, and can be done for just about any function or operator. 
* Extending the above, `.=` is different than `=`. To use  `.=` the variable on the left of the assignment must already exist and be the right size. If it does though, `.=` will be much more efficient than `=` since it will fill the already-existing array rather than allocating a new one. 
  * If you don't want to write all the dots, you can use the `@.` macro. `@. r  = a * b + sin(c)` is equivalent to `r .= a .* b .+ sin.(c)`. In either case, if you have "dot-broadcasting" for every operation in a line the whole thing will get "fused" with sometimes-significant performance gains.
* For **timing** things (in the context of optimizing performance), you generally want to use `@btime` (from [BenchmarkTools.jl](https://github.com/JuliaCI/BenchmarkTools.jl)), not `@time`, and unless you're timing something that will always be called in global scope you probably want to "interpolate" any global variables into the `@btime` macro with `$`  (e.x. `@btime somefunction($some_array)`)
* For more details, see the excellent articles on [Julia Antipatterns](https://www.oxinabox.net/2020/04/19/Julia-Antipatterns.html) by Lyndon White or [7 Julia Gotchas](http://www.stochasticlifestyle.com/7-julia-gotchas-handle/) by Chris Rackauckas

#### Performance:
(see also: https://docs.julialang.org/en/v1.5/manual/performance-tips/)
* **_Loops_** are fast in Julia. It sounds simple but it was hard to really believe / to unlearn my Matlab instincts. In particular, an `@inbounds @simd for` loop doing simple arithmetic operations, array indexing, or really anything that's *type-stable*, should run at C-like or faster speeds.
  * Type-stable, non-allocating loops are just as fast as broadcasting (if not faster!)

* **_Type-stability_**: Use `@code_warntype` followed by a function call to check for type-stability (anywhere it prints a red `Any` is bad). Type-stable code is, like, two orders of magnitude faster than type-unstable code (though even type-unstable code may be faster than Python :p). If the `@code_warntype` output is overwhelming, start by just looking at the list of variables at top and try to make sure they're all green concrete types and not red `Any`s.
  * !However: putting restrictive type assertions on function arguments doesn't make the function faster, and is more useful for dispatch than for anything to do with type stability. Instead, type-stability is all about avoiding cases where instability is introduced in the first place, and if you can't then sanitizing any unstable things (like variables pulled from a `Dict`) with a typeassert (e.g. `x::Array{Float64,1}`) prior to using them (in fact, probably better to avoid `Dict`s altogether except perhaps for interactive use)

* **_Vectorization_** is used to refer to two very different concepts in CS, both of which occur in Julia. One is a syntax thing, the other is a hardware thing. The syntax kind is what you know from Matlab (essentially what we discussed as "broadcasting" above), the hardware kind is about the amazing things you can do with your CPU's vector registers / avx extensions (basically like a mini GPU within your CPU), which you can access with [LoopVectorization.jl](https://github.com/chriselrod/LoopVectorization.jl).  
  * You can [LoopVectorization.jl](https://github.com/chriselrod/LoopVectorization.jl) either on a loop with `@avx for`, or on a dot-broadcasted operation with `@avx  @.` You can only use this in cases where the iterations of the loop can be conducted in arbitrary order, since the vector registers will be running the same operations on several iterations of your loop at the same time. 
  * See also [`@simd`](https://docs.julialang.org/en/v1.5/base/base/index.html#Base.SimdLoop.@simd)

#### Other tips:
* In Julia, you generally don't use `printf`. Instead you can interpolate variables into strings with `$` and use plain old `print` (or `println`), ex: "The values varied from $x to $y"
```julia
julia> x = 5; y = 7;
julia> print("The values varied from $x to $y")
The values varied from 5 to 7
```

##### If you want to get into the real details and see how your Julia code is being implemented in terms of machine code:
* `@code_lowered # Prints Julia SSA-form IR`
* `@code_llvm    # Prints LLVM bitcode`
* `@code_native  # Prints native assembly code`
