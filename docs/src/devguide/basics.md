# The basics

After reading this guide, you should be able to write your own geostatistical
solver, and enjoy a large set of features *for free*, including distributed
parallel execution, a suite of meta algorithms, and various plot recipes. If
you have any questions, please don't hesitate to ask in our
[gitter channel](https://gitter.im/JuliaEarth/GeoStats.jl).

Currently, there are three types of spatial problems defined in the framework:

```julia
EstimationProblem
SimulationProblem
LearningProblem
```

For each problem type, there is a corresponding solution type:

```julia
EstimationSolution
SimulationSolution
LearningSolution
```

Any `EstimationSolver` in the framework returns an `EstimationSolution`, which
consists of the domain of the `EstimationProblem` plus two dictionaries: `mdict`
mapping the variable names (a `Symbol`) of the problem to mean values, and `vdict`
mapping the same variable names to variance values:

```julia
EstimationSolution(domain, mdict, vdict)
```

Any `SimulationSolver` in the framework returns a `SimulationSolution`, which
consists of the domain of the `SimulationProblem` plus a dictionary `rdict`
mapping the variable names (a `Symbol`) of the problem to a vector of (flattened)
realizations:

```julia
SimulationSolution(domain, rdict)
```

Any `LearningSolver` in the framework returns a `LearningSolution`, which consits
of the target domain of the `LearningProblem` plus a dictionary `dict` mapping
variable names (a `Symbol`) of the problem to learned values:

```julia
LearningSolution(domain, dict)
```

These definitions are implemented in the
[GeoStatsBase.jl](https://github.com/juliohm/GeoStatsBase.jl)
package alongside other important conceptual
components of the project.

## How to write a solver

The task of writing a solver for a spatial problem as defined consists of
writing a simple function in Julia that takes the problem as input and returns
the solution. In this section, an estimation solver is written that is not very
useful (it fills the domain with random numbers), but that illustrates the
development process.

### Writing estimation solvers

#### Create the package

Install the `PkgTemplates.jl` package and create a new project:

```julia
using PkgTemplates

generate_interactive("MySolver")
```

This command will create a folder named `~/user/.julia/vx.y/MySolver` with all
the files that are necessary to load the new package:

```julia
using MySolver
```

Choose a license for your solver. If you don't have major restrictions, I suggest
using the `ISC` license. This license is equivalent to the `MIT` and `BSD 2-Clause`
licenses, plus it eliminates [unncessary language](https://en.wikipedia.org/wiki/ISC_license).
Try to choose a [permissive license](https://opensource.org/licenses) so that your
solver can be used, and improved by private companies.

#### Import GeoStatsBase

After the package is created, open the main source file `MySolver.jl` and add the
following line:

```julia
using GeoStatsBase
import GeoStatsBase: solve
```

These lines bring all the symbols defined in `GeoStatsBase` into scope, and tell
Julia that the method `solve` will be specialized for the new solver. Next, give
your solver a name:

```julia
struct MyCoolSolver <: AbstractEstimationSolver
  # optional parameters go here
end
```

and export it so that it becomes available to users of your package:

```julia
export MyCoolSolver
```

At this point, the `MySolver.jl` file should have the following content:

```julia
module MySolver

using GeoStatsBase
import GeoStatsBase: solve

export MyCoolSolver

struct MyCoolSolver <: AbstractEstimationSolver
  # optional parameters go here
end

end # module
```

#### Write the algorithm

Now that your solver type is defined, write your algorithm. Write a function called
`solve` that takes an estimation problem and your solver, and returns an estimation
solution:

```julia
function solve(problem::EstimationProblem, solver::MyCoolSolver)
  pdomain = domain(problem)

  mean = Dict{Symbol,Vector}()
  variance = Dict{Symbol,Vector}()

  for (var,V) in variables(problem)
    push!(mean, var => rand(npoints(pdomain)))
    push!(variance, var => rand(npoints(pdomain)))
  end

  EstimationSolution(pdomain, mean, variance)
end
```

Paste this function somewhere in your package, and you are all set.

#### Test the solver

To test your new solver, load the `GeoStats.jl` package and solve a simple problem:

```julia
using GeoStats
using MySolver

sdata    = readgeotable("somedata.csv", coordnames=[:x,:y])
sdomain  = RegularGrid{Float64}(100, 100)
problem  = EstimationProblem(sdata, sdomain, :value)

solution = solve(problem, MyCoolSolver())

plot(solution)
```

### Writing simulation solvers

The process of writing a simulation solver is very similar, but there is an
alternative function to `solve` called `solvesingle` that is *preferred*. The
function `solvesingle` takes a simulation problem, one of the variables to be
simulated, a solver, and a preprocessed input, and returns a *vector* with the
simulation results:

```julia
function solvesingle(problem::SimulationProblem, covars::NamedTuple,
                     solver::MySimSolver, preproc)
  # retrieve problem info
  pdata = data(problem)
  pdomain = domain(problem)

  real4var = map(covars.names) do var
    # output is a single realization for each covariable
    real = Vector{V}(undef, npoints(pdomain))

    # fill realization with hard data
    for (loc, datloc) in datamap(problem, var)
      real[loc] = pdata[datloc,var]
    end

    # algorithm goes here
    # ...

    var => real
  end

  Dict(real4var)
end
```

This function is preferred over `solve` if your algorithm is the same for every
single realization (the algorithm is only a function of the random seed). In this
case, GeoStats.jl will provide an implementation of `solve` for you that calls
`solvesingle` in parallel.

The argument `preproc` is ignored unless the function `preprocess` is also defined
for the solver. The function takes a simulation problem and a solver, and returns
an arbitrary object with preprocessed data:

```julia
preprocess(problem::SimulationProblem, solver::MySimSolver) = nothing
```

### Writing learning solvers

Similar to the other cases, writing a `LearningSolver` compatible with the framework
consists of writing a simple Julia function that takes the `LearningProblem` as input
along with the solver, and returns a `LearningSolution`.
