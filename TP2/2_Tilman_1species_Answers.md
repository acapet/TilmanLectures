Tilman’s Resource Competition : 1 species 2 resources
================
Arthur Capet
June 15, 2017

  - [Framework](#framework)
  - [Parameters](#parameters)
  - [The growth function](#the-growth-function)
  - [Time Derivatives](#time-derivatives)
  - [Dynamic simulation](#dynamic-simulation)
  - [Steady-state solution](#steady-state-solution)
  - [Growth on the resource plane](#growth-on-the-resource-plane)
  - [Trajectories](#trajectories)
  - [Perturbations](#perturbations)
  - [Exercice](#exercice)
  - [Next](#next)
  - [References](#references)

This script allows to visualize the dynamics of a single species
depending on two ressources (Tilman 1982). You might want to have a look
on the [lecture notes](https://www.overleaf.com/read/krhfddzjxnqc)
before going any further.

``` r
library("deSolve") # For solving differential equations
library("FME")     # Toolbox to play with model perturbation, sensitivity analysis, etc..
```

# Framework

Our system is defined at any given time by 3 state variables :

  - \(N_1\) (the population),
  - \(R_1\) and \(R_2\) (the ressources).

The growth of \(N_1\) will depend on the availability of both resources
in a specific way, as defined in the function `Growth`.

# Parameters

All parameters that will be used later are given in a vector (those used
in the growth function, but also those used for initial conditions,
resource supply, etc .. )

``` r
pars<-c(
  # 1 Population
  mN1  = .1    ,  # mortality N1
  ##   Params for growth
  mu1 = .5     ,  # Max Growth 
  limN1R1 = 30 ,  # Half-Saturation R1
  limN1R2 = 40 ,  # Half-Saturation R2
  a11     = .6 ,  # Resource preference for R1, by N1 [0-1]
  # Resources supply
  g1  = 10     ,  # Supply R1 (max R1)
  g2  = 40     ,  # Supply R2 (max R2)
  gT  = 10     ,  # Relaxation time towards max Conc
  # Initial conditions
  N1_0 = 10    ,  # Initial population N1
  R1_0 = 30    ,  # Initial stock R1
  R2_0 = 60    ,  # Initial stock R2
  # Simulation
  duration = 100,
  dt=.1
)
  # Ressource type
  #ftype="Essential"
  ftype="InteractiveEssential"
```

# The growth function

The main growth function `Growth` can take different form, representing
different kind of resource (cf. lectures). Currently, we’ll just switch
to one or another form by commenting/uncommenting part of the code.

`Growth` requires three arguments :

  - `R1` and `R2` are the two resources availabilities
  - `hneed` is a flag. If *`hneed=FALSE`*, the function returns only the
    growth rate. If *`hneed=TRUE`*, the function also returns `h1` and
    `h2`, the components of the resource consumption vector.

<!-- end list -->

``` r
Growth<- function (R1,R2,Pp, hneed=F) {
  # Pp gives the species parameters 
  # * limR1
  # * limR2
  # * mu 
  # The 'with' function executes the code in {} with elements of the list (first argument)
  #    included as part of the local environment
  with (as.list(Pp), {
    fR1 <- R1 / (R1 + limR1)
    fR2 <- R2 / (R2 + limR2)
    
    if (ftype %in% c("Essential",
                      "InteractiveEssential",
                      "PerfectlySubstitutive",
                      "Complementary",
                      "Antagonistic") ){
    }else{
      print(' F type unknown, imposing Essential type' )
             ftype <- "Essential"                 
                      }
    #############
    # Essential #
    #############
    if (ftype=="Essential"){
    f <- mu * pmin(fR1,fR2)
    h1 <- a
    h2 <- (1-a)
    }
    #########################
    # Interactive Essential #
    #########################
        if (ftype=="InteractiveEssential"){
    f <- mu * fR1*fR2
    a<-.2
    h1 <- (a)
    h2 <- (1-a)
    }
    ##########################
    # Perfectly Substitutive #
    ##########################
        if (ftype=="PerfectlySubstitutive"){
    f <- mu * (R1+R2)/ ( R1+R2  + limR1+ limR2 )
    h1 <- R1/(R1+R2)
    h2 <- R2/(R1+R2)
        }
     
    #################
    # Complementary #
    #################
    if (ftype=="Complementary"){
     f <- mu * ((R1+R2+R1*R2/10)/ (R1+R2+R1*R2/10+limR1+limR2))
     h1 <- R1/(R1+R2)
     h2 <- R2/(R1+R2)
    }
    ################
    # Antagonistic #
    ################
    if (ftype=="Antagonistic"){
     f <- mu * ((R1+R2-R1*R2/80)/ (R1+R2-R1*R2/80+limR1+limR2))
     h1 <- R1/(R1+R2)
     h2 <- R2/(R1+R2)
     }
    #############
    # Switching #
    #############
    # f <- mu * pmax(R1,R2)/ (pmax(R1,R2)+   limR1+limR2 )
    # 
    # h1 <- R1/(R1+R2)
    # h2 <- R2/(R1+R2)
    #  if (R1>R2){
    #  h1 <- 1
    #  } else {
    #    h1 <- 0
    #  }
    #  h2<-1-h1
    
    if (hneed){
      return(c(f=f,h1=h1,h2=h2))
    } else {
      return(f)
    }
  })
}
```

# Time Derivatives

The following function `simpleg` provides the time derivatives for the
state variables, gathered in one vector `X`, in order to compute how the
system evolves in time. Those time derivatives depend on the current
state, and are controlled by the parameters and the form of the growth
function.

``` r
simpleg <- function (t, X, parms) {
  with (as.list(parms), {
    N1 <- X[1]
    R1 <- X[2]
    R2 <- X[3]
    
        # Return the growth rate and consumption vectors for N1
    pN1<-c( limR1 = limN1R1 ,
            limR2 = limN1R2 ,
            mu    = mu1     , 
            a     = a11     )
    
    
    # Get the growth rate.
    # Because the third argument is "True", G is a vector with three named components.
    # (see definition of Growth)
    G<-Growth(R1,R2,pN1,T)
    f1<-G["f"]
    h1<-G["h1"]
    h2<-G["h2"]
    
    dN1 <- N1 * (f1 - mN1) # growth minus mortality
    dR1 <-  (g1-R1)/gT - N1 * (f1 )* h1 # supply minus consumption 
    dR2 <-  (g2-R2)/gT - N1 * (f1 )* h2 # supply minus consumption
    
    # Return the time derivative, the list structure is required by the solving package
    return(list(c(dN1, dR1 , dR2)))
  })
}
```

# Dynamic simulation

In the following, we

  - define the initial conditions `X0`,
  - define a number of time step, set a temporal framework
  - run first a dynamic simulation, solving the problem in time, ie.
    looking at the evolution of the population and resources.

<!-- end list -->

``` r
X0 <- with(as.list(pars),c(N1_0,R1_0,R2_0))

# time intervals at which we want the outputs
times <- with(as.list(pars),seq(0, duration, by = dt))

# function ode is the solver, it computes the dynamic simulation.
library(deSolve)
out <- ode(y = X0, times = times, func = simpleg, parms = pars,method = "euler")

# this give names to the simulation output 
colnames(out)<-c("time","N1","R1","R2")      
plot(out)
```

![](2_Tilman_1species_Answers_files/figure-gfm/dynamic-1.png)<!-- -->

# Steady-state solution

Next, we compute directly the steady-state solution, ie the value of
state variables for which the time derivative are nul: *growth*
compensate for *mortality*, and resource *supply* compensate for
*consumption*. The system is balanced, at equilibrium.

The values correspond to the last values of the dynamic run, but they
were computed faster, from theoretical considerations, rather than
waiting for the system to reach equilibrium by itself.

``` r
# this provides the steady state solution
outsteady<-steady(y = X0, time=c(0,Inf),func = simpleg, parms = pars, method= "runsteady")
outs <- outsteady$y
names(outs)<-c("N1","R1","R2")
print(outs)
```

    ##           N1           R1           R2 
    ## 2.795296e-07 1.000000e+01 4.000000e+01

# Growth on the resource plane

Here, we want to explore how the equilibrium point (such as obtained
above) depends on the growth function parameters and initial conditions.
The locus of different equilibirum points, in the resource place (with
axis \(R_1\) and \(R_2\)), is called the *zero net growth isoline*, or
*ZNGI*.

First, we compute growth rates for all values of \(R_1\) and \(R_2\) and
display it with colour. Second we highlight the location where growth
rate equals mortality rate.

``` r
# Defining the extent of the ressource space to explore
R1space <- seq(0,80, length=80)
R2space <- seq(0,80, length=80)

with (as.list(pars), {
  pN1 <<-c( limR1 = limN1R1 ,
          limR2 = limN1R2 ,
          mu    = mu1,
          a =a11  )})
  
# this function evaluate growth 1 for every value of (R1space,R2space)
f1space <- outer(R1space,R2space,Growth,pN1)#, ftype="InteractiveEssential")
# First we plot the contour
image(R1space ,R2space ,f1space,main="Iso-growth")

# then we add the line were growth is equal to mortality (= the value in pars["mN1"])
contour(R1space ,R2space ,f1space,levels=as.vector(pars["mN1"]),add=T,col="blue",lty = "dotted", labels="ZNGI",lwd=2)
```

![](2_Tilman_1species_Answers_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

# Trajectories

Here we will visualize the trajectory of the simulation we computed just
above.

``` r
image(R1space ,R2space ,f1space,main="Iso-growth")
contour(R1space ,R2space ,f1space,levels=as.vector(pars["mN1"]),
        add=T,col="blue",lty = "dotted", labels="ZNGI",lwd=2)

# Now, we plot R1 and R2 from the dynamic solution
lines(out[,"R1"],out[,"R2"])
# highlight the first values of R1 and R2
points(out[1,"R1"],out[1,"R2"])
# the last values
points(out[nrow(out),"R1"],out[nrow(out),"R2"],col='red')
points(outs["R1"],outs["R2"],col='red',cex=1.5)

# Locate the Supply Point
points(pars["g1"],pars["g2"],col='blue',cex=1.5,bg='blue',pch=21)
```

![](2_Tilman_1species_Answers_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

# Perturbations

Finally, to illustrate that this point is a stable equlibrium point we
will use the `modCRL` function from the FME package (Soetaert and
Petzoldt 2010), to perturbate initial conditions and display
corresponding trajectories. The modCRL function (type `?modCRL`) needs a
transfer function (`fCRL` below) that will return something according to
the parameters recieved in argument. The function `modCRL` generates
perturbation of the parameters (within some range given in `parRange`),
calls the transfer function for each parameter setand stores the result.
In the present case, it is the transfer function `fCRL` that directly
display the trajectory on the plot.

``` r
# Just reproduce the same plot as above
image(R1space ,R2space ,f1space,main="Iso-growth")
contour(R1space ,R2space ,f1space,levels=as.vector(pars["mN1"]),
        add=T,col="blue",lty = "dotted", labels="ZNGI",lwd=2)
points(pars["g1"],pars["g2"],col='blue',cex=1.5,bg='blue',pch=21)

# define the function to be used by modCRL  
fCRL<-function(parinit){
    X0 <- with(as.list(parinit),c(N1_0,R1_0,R2_0))
    out<- ode(y = X0, times = times, func = simpleg, parms = pars,method="euler")
    colnames(out)<-c("time","N1","R1","R2")
    
    lines(out[,"R1"],out[,"R2"]  )
    points(out[1,"R1"],out[1,"R2"])
    points(out[nrow(out),"R1"],out[nrow(out),"R2"],col='red')
    return(c("R1eq"=out[nrow(out),"R1"], "R2eq"=out[nrow(out),"R2"]))
  }
  
# define the perturbation of initial conditions in the matrix parRange
# nr and nc are number of rows and columns,
# next vector the values (of min and max)
# dimnames give names to columns and rows of the matric
parRange <- matrix(nr = 3, nc = 2, 
                   c(0.2, 10,10,50,80,80),
                     dimnames = list(c("N1_0","R1_0","R2_0"), c("min", "max")))

# calling fCRL for 20 set of parameters whitin the range parRange
CRL<-modCRL(fCRL,parRange=parRange,num = 20)
```

![](2_Tilman_1species_Answers_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->

# Exercice

The objective is to see how the supply of ressources determines the
equilibrium points and the size of population at equilibirum. So,
instead of perturbating the initial conditions, we’ll perturbate the
position of the supply point.

  - Copy the last code chunk below
  - Which parameters should be perturbated ? -\> modify `parRange`
    accordingly.

Inside the function `fCRL` \* Removes the line changing the initial
condition. The `X0` global value will be used instead. \* Copy the
global parameter vector `pars` in a local perturbated parameter vector.
\* Replace the value of perturbated parameters by those received in
argument \* use this perturbated parameter vector instead of the global
`pars` when computing the model solution.

For the plot: \* add a blue point for the perturbated supply point, give
it a size that depends on the size of population at equilibirum (use
`cex = "N_eq"/20`). \* add a blue line between the supply point and the
reached equilibrium

``` r
# Just reproduce the same plot as above
image(R1space ,R2space ,f1space,main="Iso-growth")
contour(R1space ,R2space ,f1space,levels=as.vector(pars["mN1"]),
        add=T,col="blue",lty = "dotted", labels="ZNGI",lwd=2)

# define the function to be used by modCRL  
fCRL<-function(parloc){
  print(' 1. parloc: ')
  print(parloc)
    perturbpars<-pars
    perturbpars[names(parloc)]<-parloc
    out<- ode(y = X0, times = times, func = simpleg, parms = perturbpars,method="euler")
    colnames(out)<-c("time","N1","R1","R2")
    
    lines(out[,"R1"],out[,"R2"]  )
    points(out[1,"R1"],out[1,"R2"])
#    points(out[nrow(out),"R1"],out[nrow(out),"R2"],col='green',pch=21,)
    points(perturbpars["g1"],perturbpars["g2"],col='blue',bg='blue',pch=21,cex=out[nrow(out),"N1"]/20 )
    lines(c(perturbpars["g1"],out[nrow(out),"R1"]),c(perturbpars["g2"],out[nrow(out),"R2"]) ,col='blue' )

    return(c("R1eq"=out[nrow(out),"R1"], "R2eq"=out[nrow(out),"R2"]))
  }
  
# define the perturbation of Ressource Supply Point in the matrix parRange
# nr and nc are number of rows and columns,
# next vector the values (of min and max)
# dimnames give names to columns and rows of the matric
parRange <- matrix(nr = 2, nc = 2, 
                   c(5,5,80,80),
                     dimnames = list(c("g1","g2"), c("min", "max")))

# calling fCRL for 20 set of parameters whitin the range parRange
CRL<-modCRL(fCRL,parRange=parRange,num = 20)
```

![](2_Tilman_1species_Answers_files/figure-gfm/unnamed-chunk-8-1.png)<!-- -->

    ## [1] " 1. parloc: "
    ##   g1   g2 
    ## 42.5 42.5 
    ## [1] " 1. parloc: "
    ##   g1   g2 
    ## 42.5 42.5 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 33.17302 24.36469 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 68.29161 18.54391 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 22.21515 35.62545 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 65.57154 51.78793 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 47.23258 79.33538 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 31.95666 52.19107 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 46.79992 35.61007 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 33.21345 16.64379 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 42.41520 13.94377 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 34.45076 26.14552 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 63.35701 19.69039 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 69.64201 65.64855 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 58.73063 39.26778 
    ## [1] " 1. parloc: "
    ##      g1      g2 
    ## 30.3947 14.0553 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 17.22593 74.84607 
    ## [1] " 1. parloc: "
    ##        g1        g2 
    ## 48.053398  9.573051 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 11.47255 24.42736 
    ## [1] " 1. parloc: "
    ##        g1        g2 
    ##  5.994428 72.944885 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 39.05472 79.38052 
    ## [1] " 1. parloc: "
    ##       g1       g2 
    ## 28.97189 73.84300

# Next

Next we will see what happens when two species competes for the same
resources : [the 2 species case](3_Tilman_2species.pdf)

# References

<div id="refs" class="references hanging-indent">

<div id="ref-SOETAERTFME">

Soetaert, Karline, and Thomas Petzoldt. 2010. “Inverse modelling,
sensitivity and monte carlo analysis in R using package FME.” *Journal
of Statistical Software* 33 (3): 1–28.

</div>

<div id="ref-TILMAN">

Tilman, David. 1982. *Resource Competition and Community Structure*.
Princeton university press.

</div>

</div>
