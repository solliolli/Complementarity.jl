# Complementarity.jl

[![Build Status](https://travis-ci.org/chkwon/Complementarity.jl.svg?branch=master)](https://travis-ci.org/chkwon/Complementarity.jl)
[![Coverage Status](https://coveralls.io/repos/github/chkwon/Complementarity.jl/badge.svg?branch=master)](https://coveralls.io/github/chkwon/Complementarity.jl?branch=master)


This package provides a modeling and computational interface for solving Mixed Complementarity Problems (MCP): modeling by [JuMP.jl](https://github.com/JuliaOpt/JuMP.jl) and computing by [PATHSolver.jl](https://github.com/chkwon/PATHSolver.jl).

The form of MCP is as follows:
```
lb ≤ x ≤ ub ⟂ F(x)
```
which means
- `x = lb`, then `F(x) ≥ 0`
- `lb < x < ub`, then `F(x) = 0`
- `x = ub`, then `F(x) ≤ 0`
When there is no upper bound `ub`, and the lower bound `lb=0`, then it is a regular Nonlinear Complementarity Problem (NCP) of the form:
```
0 ≤ x ⟂ F(x) ≥ 0
```
which means
```
F(x)' x = 0, F(x) ≥ 0, x ≥ 0
```
When `F(x)` is a linear operator such as `F(x) = M x + q` with matrix `M` and vector `q`, then it is a Linear Complementarity Problem (LCP). All these problems are solved by the [PATH Solver](http://pages.cs.wisc.edu/%7Eferris/path.html) which is wrapped by the [PATHSolver.jl](https://github.com/chkwon/PATHSolver.jl) package.

This package `Complementarity.jl` extends the modeling language from [JuMP.jl](https://github.com/JuliaOpt/JuMP.jl) to model complementarity problems.


# OS X only

At this moment, `PATHSolver.jl` is only available for Mac OS X, and its installation requires some work. Because this package `Complementarity.jl` is dependent on `PATHSolver.jl`, it also is available for Mac OS X only.

# License

The PATH Solver requires a license. The web page of the PATH Solver provides a temporary license that is valid for a year. A new license is provided each year in the web page. Visit the [license page](http://pages.cs.wisc.edu/~ferris/path/LICENSE) of the PATH Solver.

Edit your `.bash_profile` file. For example, if you have `Atom` editor:
```bash
atom ~/.bash_profile
```
and add the following two lines:
```bash
export PATH_LICENSE_STRING="---------------------------------------------------------------"
```
You can obtain the most recent `PATH_LICENSE_STRING` from [the website of the PATH Solver](http://pages.cs.wisc.edu/~ferris/path/LICENSE). To reflect the change:
```bash
source ~/.bash_profile
```

# Installation

```julia
Pkg.add("Complementarity")
```

If an error occurs during installation, it is most likely a problem in the `PATHSolver.jl` package. Try first to build `PATHSolver.jl` by doing `Pkg.build("PATHSolver")`.

You may need to install 'Xcode Command Line Tools'. On Mavericks or higher (Yosemite, El Capitan), you can install it by entering the following command in your terminal:
```
xcode-select --install
```
For older versions of Mac OS X, see [this post](http://mac-how-to.wonderhowto.com/how-to/install-command-line-developer-tools-without-xcode-0168115/).


# Example 1

```julia
m = MCPModel()

M = [0  0 -1 -1 ;
     0  0  1 -2 ;
     1 -1  2 -2 ;
     1  2 -2  4 ]

q = [2; 2; -2; -6]

lb = zeros(4)
ub = Inf*ones(4)

items = 1:4

# @defVar(m, lb[i] <= x[i in items] <= ub[i])
@defVar(m, x[i in items] >= 0)
@defNLExpr(m, F[i in items], sum{M[i,j]*x[j], j in items} + q[i])
correspond(m, F, x)

PATHSolver.path_options(
                "convergence_tolerance 1e-2",
                "output no",
                "time_limit 3600"
                )

solveMCP(m)

z = getValue(x)
````
The result should be `[2.8, 0.0, 0.8, 1.2]`.

```julia
m = MCPModel()
```
This line prepares a JuMP Model, just same as in [JuMP.jl](https://github.com/JuliaOpt/JuMP.jl).

```julia
@defVar(m, x[i in items] >= 0)
```
Defining variables is exactly same as in JuMP.jl. Lower and upper bounds on the variables in the MCP model should be provided here.

```julia
@defNLExpr(m, F[i in items], sum{M[i,j]*x[j], j in items} + q[i])
```
This is to define expressions for `F` in MCP. Even when the expression is linear or quadratic, you should use the nonlinear version `@defNLExpr`.

```julia
correspond(m, F, x)
```
This function matches each element of `F` and the corresponding element of `x`.

```julia
PATHSolver.path_options(   
                "convergence_tolerance 100",
                "output no",
                "time_limit 3600"      )
```
This adjusts options of the PATH Solver. See the [list of options](http://www.cs.wisc.edu/~ferris/path/options.pdf).

```julia
solveMCP(m)
```
This solves the MCP and stores the solution inside `m`, which can be accessed by `getValue(x)` as in JuMP.


# Example 2

This is a translation of [`transmcp.gms`](http://www.gams.com/modlib/libhtml/transmcp.htm) originally written in GAMS.

```julia
plants = ["seattle", "san-diego"]
markets = ["new-york", "chicago", "topeka"]

capacity = [350, 600]
a = Dict(zip(plants, capacity))

demand = [325, 300, 275]
b = Dict(zip(markets, demand))

elasticity = [1.5, 1.2, 2.0]
esub = Dict(zip(markets, elasticity))

distance = [ 2.5 1.7 1.8 ;
             2.5 1.8 1.4  ]
d = Dict()
for i in 1:length(plants), j in 1:length(markets)
    d[plants[i], markets[j]] = distance[i,j]
end

f = 90

using Complementarity, JuMP

m = MCPModel()
@defVar(m, w[i in plants] >= 0)
@defVar(m, p[j in markets] >= 0)
@defVar(m, x[i in plants, j in markets] >= 0)

@defNLExpr(m, c[i in plants, j in markets], f * d[i,j] / 1000)

@defNLExpr(m, profit[i in plants, j in markets], w[i] + c[i,j] - p[j])
@defNLExpr(m, supply[i in plants], a[i] - sum{x[i,j], j in markets})
@defNLExpr(m, fxdemand[j in markets], sum{x[i,j], i in plants} - b[j])

correspond(m, profit, x)
correspond(m, supply, w)
correspond(m, fxdemand, p)

PATHSolver.path_options(
                "convergence_tolerance 1e-2",
                "output no",
                "time_limit 3600"
                )

solveMCP(m)

@show getValue(x)
@show getValue(w)
@show getValue(p)
```

The result is
```julia
getValue(x) = x: 2 dimensions:
[  seattle,:]
  [  seattle,new-york] = 67.70981462842977
  [  seattle, chicago] = 1175.6096055006494
  [  seattle,  topeka] = 2389.1555381352864
[san-diego,:]
  [san-diego,new-york] = 198.46702743401073
  [san-diego, chicago] = 793.3418797392139
  [san-diego,  topeka] = 1245.136516087478

getValue(w) = w: 1 dimensions:
[  seattle] = 72.21062818239575
[san-diego] = 152.84461370674097

getValue(p) = p: 1 dimensions:
[new-york] = 68.04769346598324
[ chicago] = 5.747377724425455
[  topeka] = 0.0

p: 1 dimensions:
[new-york] = 68.04769346598324
[ chicago] = 5.747377724425455
[  topeka] = 0.0
```
