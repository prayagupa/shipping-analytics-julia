
Producer-consumer scheme
-----------------------------

Tasks as iterators

```

julia> weightPackages = @task { for weight in [1,2,4] println(string("weight : ", weight)) end }

WARNING: deprecated syntax "{a,b, ...}".
Use "Any[a,b, ...]" instead.
Task (runnable) @0x000000010adc8a90


julia> istaskdone(weightPackages)
false

julia> current_task()
Task (waiting) @0x0000000107361f90

julia> consume(weightPackages)
1
2
4
1-element Array{Any,1}:
 nothing

julia> consume(weightPackages)
1-element Array{Any,1}:
 nothing

julia> consume(weightPackages)
1-element Array{Any,1}:
 nothing

```

Parallel and Distributed Ccomputing with Julia

julia remotecall
-------------------

```
$ julia -p 4
               _
   _       _ _(_)_     |  A fresh approach to technical computing
  (_)     | (_) (_)    |  Documentation: http://docs.julialang.org
   _ _   _| |_  __ _   |  Type "?help" for help.
  | | | | | | |/ _` |  |
  | | |_| | | | (_| |  |  Version 0.4.5 (2016-03-18 00:58 UTC)
 _/ |\__'_|_|_|\__'_|  |  Official http://julialang.org/ release
|__/                   |  x86_64-apple-darwin13.4.0


julia> processorNumber = 2

julia> remoteRefToMatrix = remotecall(processorNumber, rand, 2, 2)
RemoteRef{Channel{Any}}(2,1,5)

# fetch can be considered an explicit data movement operation, since it directly
# asks that an object be moved to the local machine

julia> fetch(remoteRefToMatrix)
2x2 Array{Float64,2}:
 0.966868  0.335636 
 0.55773   0.0510259

julia> addOneToMatrix = @spawnat processorNumber 1+fetch(remoteRefToMatrix)
RemoteRef{Channel{Any}}(2,1,7)

julia> fetch(addOneToMatrix)
2x2 Array{Float64,2}:
 1.96687  1.33564
 1.55773  1.05103

julia> remotecall_fetch(processorNumber, getindex, remoteRefToMatrix, 1, 1)
0.966868
```


```
julia> grades = @spawn rand(2, 2)
RemoteRef{Channel{Any}}(2,1,13)

julia> addGraceToGrades = @spawn 1+fetch(grades)
RemoteRef{Channel{Any}}(3,1,14)

julia> fetch(addGraceToGrades)
2x2 Array{Float64,2}:
 1.33169  1.28137
 1.11412  1.12581

julia> fetch(grades)
2x2 Array{Float64,2}:
 0.331693  0.28137 
 0.114122  0.125808
```

Availability of a function to processors
------------------------------------------

```
# method 1 - a random matrix is constructed locally, then sent to another processor where it is squared.
# good when : matrix data initialization is expensive operation
songsMatrix_YearByMonths = rand(1000, 1000)
# matrix data initialization can also be parallelized with @spawn
# songsMatrix_YearByMonths = @spawn rand(1000, 1000)

squareMatrix = @spawn songsMatrix_YearByMonths^2

fetch(squareMatrix)

##method 2 -  a random matrix is both constructed and squared on another processor
julia> squareMatrix = @spawn rand(1000, 1000)^2
RemoteRef{Channel{Any}}(4,1,30)
```


```

julia> addprocs(3)
3-element Array{Int64,1}:
 6
 7
 8

## sequential

julia> @everywhere function fibonacci(number)
                        if( number < 2 ) then 
                           return number
                        else
                           return fibonacci(number-1) + fibonacci(number-2)
                        end
                    end

julia> someFibonacci = @spawn fibonacci(10)
RemoteRef{Channel{Any}}(3,1,41)

julia> fetch(someFibonacci)
55

julia> @time [fibonacci(number) for number=1:45]
 25.220634 seconds (1 allocation: 448 bytes)
45-element Array{Int64,1}:

## parallel

julia> @everywhere function fibonacciInParallel(number)
                        if( number < 40 ) then 
                           return fibonacci(number)
                        else
                           newFibProcess1 = @spawn fibonacciInParallel(number-1)
                           newFibProcess2 = fibonacciInParallel(number-2)
                           return fetch(newFibProcess1) + newFibProcess2 
                        end
                   end

julia> @time [fibonacciInParallel(number) for number=1:45]
 10.786086 seconds (4.52 k allocations: 305.465 KB)
45-element Array{Any,1}:

```

parallel reduction using @parallel foreach
---------------------------------------------
 
```
julia> items = zeros(4)
4-element Array{Float64,1}:
 0.0
 0.0
 0.0
 0.0

julia> @parallel for iterationIndex = 1:4 
                       items[iterationIndex] = iterationIndex
                 end
4-element Array{Any,1}:
 RemoteRef{Channel{Any}}(2,1,81)
 RemoteRef{Channel{Any}}(3,1,82)
 RemoteRef{Channel{Any}}(4,1,83)
 RemoteRef{Channel{Any}}(5,1,84)

# Iterations run on different processors and do not happen in a specified order,
# Conseqnently, variables or arrays will not be globally visible.
# Any variables used inside the parallel loop will be copied and broadcast to each processor.

# Processors produce results which are made visible to the launching processor via the reduction.

julia> items
4-element Array{Float64,1}:
 0.0
 0.0
 0.0
 0.0

julia> for seqIndex = 1:4
               items[seqIndex] = seqIndex
       end

julia> items
4-element Array{Float64,1}:
 1.0
 2.0
 3.0
 4.0
```
