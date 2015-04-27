
#Lalonde example

Throughout the user's guide, we'll use the [Lalonde dataset](http://users.nber.org/~rdehejia/nswdata2.html), in particular the "Dehejia-Wahha Sample" on that web page. It is originally from the paper
Robert Lalonde, "Evaluating the Econometric Evaluations of Training Programs," *American Economic Review*, Vol. 76, pp. 604-620. 1986.

The output in this example was generated by converting an IJulia notebook to Markdown for the user's guide. This doesn't look great, but you can find a [better looking version rendered by nbviewer](http://nbviewer.ipython.org/url/lendle.github.io/TargetedLearning.jl/user-guide/lalonde_example.ipynb).

#Getting the data set

To download the example data set, run

```julia
include(joinpath(Pkg.dir("TargetedLearning"), "examples", "fetchdata.jl"))
fetchdata(ds="lalonde_dw")
```

This only needs to be run once.

# Loading the data set

## Using readcsv

The simplest way to read the data set is with `readcsv`. The CSV file we have has a header, so we'll pass `header=true`, and we'll get back a tuple containing a matrix with the numeric data in it (`readcsv` can figure out that it's all `Float64`s automatically) and a matrix with one row containing column names.


    dsfname = joinpath(Pkg.dir("TargetedLearning"), "examples", "data", "lalonde_dw.csv")
    
    dat, colnames = readcsv(dsfname, header=true)
    
    #lets inspect colnames
    colnames




    1x10 Array{String,2}:
     "treatment"  "age"  "education"  …  "nodegree"  "RE74"  "RE75"  "RE78"




    #convert colnames to a vector instead of a matrix with one row
    colnames = reshape(colnames, size(colnames, 2))




    10-element Array{String,1}:
     "treatment"
     "age"      
     "education"
     "black"    
     "hispanic" 
     "married"  
     "nodegree" 
     "RE74"     
     "RE75"     
     "RE78"     



`treatment` is obviously the treatment variable. The outcome variable is `RE78` (earnings in 1978), and `RE74` and `RE75` are earnings values prior to treatment. The others are potential baseline confounders. Check the link above for more information about the dataset.  We want to slice the matrix `dat` up to extract the treatment and outcome variable. Julia uses square brackets for indexing. The first dimension of a matrix is rows and the second is columns like R. If you want everything in one dimension, you put `:` (where as in R you can leave that dimension empty). You can index with booleans, integers, vectors of integers, ranges and some other things. Here are some examples:


    #we know column 1 is treatment, so get it directly, and ask for all rows with :
    treatment = dat[:, 1]




    445-element Array{Float64,1}:
     1.0
     1.0
     1.0
     1.0
     1.0
     1.0
     1.0
     1.0
     1.0
     1.0
     1.0
     1.0
     1.0
     ⋮  
     0.0
     0.0
     0.0
     0.0
     0.0
     0.0
     0.0
     0.0
     0.0
     0.0
     0.0
     0.0




    #suppose instead we want to find the position in colnames that has the value
    #"treatment", but we don't know it's the first one. There are a couple of ways
    #to do that.
    #(.== and operators starting with `.` in general indicate that we want to do
    #   an element wise operation)
    
    #(the ; at the end of the line suppresses output)
    treatment_take2 = dat[:, colnames .== "treatment", 1];


    #the last column is the outcome so we can use the keyword `end`
    outcome = dat[:, end]
    
    #we can also use `end` in arithmetic, e.g.
    outcome_prebaseline = dat[:, end-1]




    445-element Array{Float64,1}:
         0.0 
         0.0 
         0.0 
         0.0 
         0.0 
         0.0 
         0.0 
         0.0 
         0.0 
         0.0 
         0.0 
         0.0 
         0.0 
         ⋮   
      9160.69
      9210.45
      9311.94
      9319.44
     10033.9 
     10598.7 
     10857.2 
     12357.2 
     13371.3 
     16341.2 
     16946.6 
     23032.0 




    #baseline covariates are everything between the first and last column, so we could use a range:
    baseline_covars = dat[:, 2:end-1]




    445x8 Array{Float64,2}:
     37.0  11.0  1.0  0.0  1.0  1.0      0.0        0.0 
     22.0   9.0  0.0  1.0  0.0  1.0      0.0        0.0 
     30.0  12.0  1.0  0.0  0.0  0.0      0.0        0.0 
     27.0  11.0  1.0  0.0  0.0  1.0      0.0        0.0 
     33.0   8.0  1.0  0.0  0.0  1.0      0.0        0.0 
     22.0   9.0  1.0  0.0  0.0  1.0      0.0        0.0 
     23.0  12.0  1.0  0.0  0.0  0.0      0.0        0.0 
     32.0  11.0  1.0  0.0  0.0  1.0      0.0        0.0 
     22.0  16.0  1.0  0.0  0.0  0.0      0.0        0.0 
     33.0  12.0  0.0  0.0  1.0  0.0      0.0        0.0 
     19.0   9.0  1.0  0.0  0.0  1.0      0.0        0.0 
     21.0  13.0  1.0  0.0  0.0  0.0      0.0        0.0 
     18.0   8.0  1.0  0.0  0.0  1.0      0.0        0.0 
      ⋮                         ⋮                       
     29.0   9.0  1.0  0.0  0.0  1.0   9268.94    9160.69
     28.0   9.0  1.0  0.0  1.0  1.0  10222.4     9210.45
     30.0  11.0  1.0  0.0  1.0  1.0      0.0     9311.94
     25.0  10.0  1.0  0.0  1.0  1.0  13520.0     9319.44
     28.0  11.0  1.0  0.0  1.0  1.0    824.389  10033.9 
     22.0  10.0  0.0  0.0  0.0  1.0  27864.4    10598.7 
     44.0   9.0  1.0  0.0  1.0  1.0  12260.8    10857.2 
     21.0   9.0  1.0  0.0  0.0  1.0  31886.4    12357.2 
     28.0  11.0  1.0  0.0  0.0  1.0  17491.5    13371.3 
     29.0   9.0  0.0  1.0  0.0  1.0   9594.31   16341.2 
     25.0   9.0  1.0  0.0  1.0  1.0  24731.6    16946.6 
     22.0  10.0  0.0  0.0  1.0  1.0  25720.9    23032.0 



## Using DataFrame

The `readtable` function returns a `DataFrame` which allows for indexing by column names.


    using DataFrames #`using` is like `library` in R
    
    dsfname = joinpath(Pkg.dir("TargetedLearning"), "examples", "data", "lalonde_dw.csv")
    
    df = readtable(dsfname)
    
    #check the column names:
    names(df)




    10-element Array{Symbol,1}:
     :treatment
     :age      
     :education
     :black    
     :hispanic 
     :married  
     :nodegree 
     :RE74     
     :RE75     
     :RE78     



Data frames in Julia are indexed by symbols instead of strings. Symbols are no entirely unlike strings, and are created in julia with `:symbolname` or `Symbol("symbolname")`.

Now let's get the treatment and outcome variables out of `df`.


    treatment = df[:treatment]
    outcome = df[:RE78]




    445-element DataArray{Float64,1}:
      9930.05
      3595.89
     24909.5 
      7506.15
       289.79
      4056.49
         0.0 
      8472.16
      2164.02
     12418.1 
      8173.91
     17094.6 
         0.0 
         ⋮   
         0.0 
      1239.84
      3982.8 
         0.0 
         0.0 
      7094.92
     12359.3 
         0.0 
         0.0 
     16900.3 
      7343.96
      5448.8 



We see that variables in a data frame are actually `DataArray`s instead of regular Julia `Array`s. The functions in TargetedLearning.jl currently only work with `Arrays` of floating point numbers, so we'll convert them.



    treatment = convert(Array{Float64}, treatment)
    outcome = convert(Array{Float64}, outcome)
    typeof(outcome)




    Array{Float64,1}



We can also index into the data frame using ranges, just like regular matrixes (but we only index on columns). Let's get the baseline covariates. When you get more than one column out of a data frame, you get back another data frame. For some reason that I do not know, `convert` won't work for us here, but `array` will get us what we need (a Julia array instead of a DataFrame).


    baseline_covars = array(df[2:end-1])




    445x8 Array{Real,2}:
     37  11  1  0  1  1      0.0        0.0 
     22   9  0  1  0  1      0.0        0.0 
     30  12  1  0  0  0      0.0        0.0 
     27  11  1  0  0  1      0.0        0.0 
     33   8  1  0  0  1      0.0        0.0 
     22   9  1  0  0  1      0.0        0.0 
     23  12  1  0  0  0      0.0        0.0 
     32  11  1  0  0  1      0.0        0.0 
     22  16  1  0  0  0      0.0        0.0 
     33  12  0  0  1  0      0.0        0.0 
     19   9  1  0  0  1      0.0        0.0 
     21  13  1  0  0  0      0.0        0.0 
     18   8  1  0  0  1      0.0        0.0 
      ⋮               ⋮                     
     29   9  1  0  0  1   9268.94    9160.69
     28   9  1  0  1  1  10222.4     9210.45
     30  11  1  0  1  1      0.0     9311.94
     25  10  1  0  1  1  13520.0     9319.44
     28  11  1  0  1  1    824.389  10033.9 
     22  10  0  0  0  1  27864.4    10598.7 
     44   9  1  0  1  1  12260.8    10857.2 
     21   9  1  0  0  1  31886.4    12357.2 
     28  11  1  0  0  1  17491.5    13371.3 
     29   9  0  1  0  1   9594.31   16341.2 
     25   9  1  0  1  1  24731.6    16946.6 
     22  10  0  0  1  1  25720.9    23032.0 



### Formulas

One nice thing about DataFrames.jl is that is has support for formulas. They are similar to R's formulas, but there are some differences. Some packages, like GLM.jl take data frames with formulas as input. Those packages can be used for computing initial estimates, but TargetedLearning.jl does not support formulas and DataFrames directly.  However, you can use DataFrames.jl's functionality to take a DataFrame and a formula and get a numeric design matrix based on a formula.

For example, suppose we'd like to include an age squared term and an interaction term between education and marital status. It looks like polynomial terms aren't implemented currently, so you'll have to manually make those terms.


    #create age squared
    df[:age2] = df[:age] .* df[:age];
    
    #if you want to suppress the intercept, use with -1
    # by default the first column will be a column of 1s, which we want.
    fm = treatment ~ age + age2 + education + black + hispanic + married + nodegree + RE74 + RE75 + married&nodegree




    Formula: treatment ~ age + age2 + education + black + hispanic + married + nodegree + RE74 + RE75 + married & nodegree




    #take the field named `m` from the created ModelMatrix object
    expanded_baseline_covars = ModelMatrix(ModelFrame(fm, df)).m




    445x11 Array{Float64,2}:
     1.0  37.0  1369.0  11.0  1.0  0.0  1.0  1.0      0.0        0.0   1.0
     1.0  22.0   484.0   9.0  0.0  1.0  0.0  1.0      0.0        0.0   0.0
     1.0  30.0   900.0  12.0  1.0  0.0  0.0  0.0      0.0        0.0   0.0
     1.0  27.0   729.0  11.0  1.0  0.0  0.0  1.0      0.0        0.0   0.0
     1.0  33.0  1089.0   8.0  1.0  0.0  0.0  1.0      0.0        0.0   0.0
     1.0  22.0   484.0   9.0  1.0  0.0  0.0  1.0      0.0        0.0   0.0
     1.0  23.0   529.0  12.0  1.0  0.0  0.0  0.0      0.0        0.0   0.0
     1.0  32.0  1024.0  11.0  1.0  0.0  0.0  1.0      0.0        0.0   0.0
     1.0  22.0   484.0  16.0  1.0  0.0  0.0  0.0      0.0        0.0   0.0
     1.0  33.0  1089.0  12.0  0.0  0.0  1.0  0.0      0.0        0.0   0.0
     1.0  19.0   361.0   9.0  1.0  0.0  0.0  1.0      0.0        0.0   0.0
     1.0  21.0   441.0  13.0  1.0  0.0  0.0  0.0      0.0        0.0   0.0
     1.0  18.0   324.0   8.0  1.0  0.0  0.0  1.0      0.0        0.0   0.0
     ⋮                             ⋮                                   ⋮  
     1.0  29.0   841.0   9.0  1.0  0.0  0.0  1.0   9268.94    9160.69  0.0
     1.0  28.0   784.0   9.0  1.0  0.0  1.0  1.0  10222.4     9210.45  1.0
     1.0  30.0   900.0  11.0  1.0  0.0  1.0  1.0      0.0     9311.94  1.0
     1.0  25.0   625.0  10.0  1.0  0.0  1.0  1.0  13520.0     9319.44  1.0
     1.0  28.0   784.0  11.0  1.0  0.0  1.0  1.0    824.389  10033.9   1.0
     1.0  22.0   484.0  10.0  0.0  0.0  0.0  1.0  27864.4    10598.7   0.0
     1.0  44.0  1936.0   9.0  1.0  0.0  1.0  1.0  12260.8    10857.2   1.0
     1.0  21.0   441.0   9.0  1.0  0.0  0.0  1.0  31886.4    12357.2   0.0
     1.0  28.0   784.0  11.0  1.0  0.0  0.0  1.0  17491.5    13371.3   0.0
     1.0  29.0   841.0   9.0  0.0  1.0  0.0  1.0   9594.31   16341.2   0.0
     1.0  25.0   625.0   9.0  1.0  0.0  1.0  1.0  24731.6    16946.6   1.0
     1.0  22.0   484.0  10.0  0.0  0.0  1.0  1.0  25720.9    23032.0   1.0



It's clunky, but will get the job done. More detailed documentation is [here](http://dataframesjl.readthedocs.org/en/latest/formulas.html).

### Missing data and categorical data

[DataFrames.jl](http://dataframesjl.readthedocs.org/en/latest/) and [DataArrays.jl](https://github.com/JuliaStats/DataArrays.jl) have ways of handling both missing data and categorical data. TargetedLearning.jl does not, so you'll have to deal with those issues ahead of time. The documentation for those packages has more information on both.

# TMLE for the average treatment effect

To use TargetedLearning.jl, we first need to scale the outcome to be between 0 and 1.


    scaled_outcome = (outcome .- minimum(outcome)) ./ (maximum(outcome) - minimum(outcome))
    extrema(scaled_outcome)




    (0.0,1.0)



We now need to estimate $\bar{Q}_0$ and $g_0$.  We'll use logistic regression for both, regressing on the columns created using the formula [above](#Formulas).


    using TargetedLearning
    
    n = size(expanded_baseline_covars, 1)
    
    #estimate \bar{Q} by regressing on baseline covars and treatment
    Qfit = lreg([expanded_baseline_covars treatment], scaled_outcome)
    
    #computed estimates of logit(\bar{Q}_n(a,W) for a=1 and 0
    #linpred returns the linear part of the prediction, which is in the logit scale for logistic regression
    logitQnA1 = linpred(Qfit, [expanded_baseline_covars ones(n)])
    logitQnA0 = linpred(Qfit, [expanded_baseline_covars zeros(n)])
    
    #esitmate g by regressing on W
    gfit = lreg(expanded_baseline_covars, treatment)
    #compute estimated probabilities of treatment
    gn1 = predict(gfit, expanded_baseline_covars);

    Warning: could not import Base.add! into NumericExtensions


Given initial estimates $\bar{Q}_n$ and $g_n$, we just need to call the `tmle` function:


    scaled_estimate = tmle(logitQnA1, logitQnA0, gn1, treatment, scaled_outcome, param=ATE(), weightedfluc=false)




    Estimate
          Estimate Std. Error Lower 95% CL Upper 95% CL
    ATE  0.0271417  0.0112603   0.00507195    0.0492115




The `tmle` function returns an estimate along with an estimated variance based on the EIC. It's important to note that this variance is asymptotically consistent if both $\bar{Q}_n$ and $g_n$ are consistent, and is conservative if $g_n$ is consistent but $\bar{Q}_n$ is not. If $g_n$ is not consistent, then EIC based variance estimate is not reliable.

The outcome in the original data set represented earnings, but this estimate is for the ATE of an outcome scaled between 0 and 1. To get it back to the original scale, we can just multiply by the scaling factor.


    estimate_orig_scale = scaled_estimate * (maximum(outcome) - minimum(outcome))




    Estimate
         Estimate Std. Error Lower 95% CL Upper 95% CL
    psi   1636.86    679.085      305.879      2967.84




After transforming the original estimate, the estimand name is now called "psi" instead of "ATE". To make the output look nicer, we can rename it:


    name!(estimate_orig_scale, "ATE on earnings")




    Estimate
                     Estimate Std. Error Lower 95% CL Upper 95% CL
    ATE on earnings   1636.86    679.085      305.879      2967.84




# CTMLE for the average treatment effect

To use CTMLE to estimate the ATE, we'll use the same initial estimate of $\mbox{logit}\bar{Q}_n$ as above, stored in `logitQnA1` and `logitQnA0`.  We'll also use the same covariates created by [expanding](#Formulas) the original variables in the data set via a formula, with a first column of ones.

First, we create a CV plan which we can use for multiple calls to `ctmle`, so results don't change based on different CV folds.  By default, the CV plan is generated inside the `ctmle` function, and can be different if you don't reset the seed random seed before each run.


    srand(20) #set the random seed
    
    #collect the treatment indexes from the StratifiedKfold object. 
    cvplan = collect(StratifiedKfold(treatment, 10)) #10 fold cv stratified by treatment;


    ctmle_est = ctmle(logitQnA1, logitQnA0, expanded_baseline_covars, treatment, scaled_outcome, cvplan=cvplan)




    Estimate
         Estimate Std. Error Lower 95% CL Upper 95% CL
    ATE  0.027733  0.0108727   0.00642292    0.0490431




To see what covariates were added in fluctuations, call `flucinfo` on the estimate:


    flucinfo(ctmle_est)




    Fluctuation info:
    Covariates added in 1 steps to 1 fluctuations.
    Fluc 1 covars added: [1]




In this case, there is only one fluctuation, and $g_0$ is only estimated using the intercept. As above, we can rescale the estimate back to the original scale. 


    scaled_estimate * (maximum(outcome) - minimum(outcome))




    Estimate
         Estimate Std. Error Lower 95% CL Upper 95% CL
    psi   1636.86    679.085      305.879      2967.84




Let's see what happens with an unadjusted initial $\bar{Q}_n$. Here we'll just use the mean of the outcome by treatment. 


    unadjusted_logitQnA1 = fill(logit(mean(scaled_outcome[treatment.==1])), n)
    unadjusted_logitQnA0 = fill(logit(mean(scaled_outcome[treatment.==0])), n)
    unadjusted_ctmle_est = ctmle(unadjusted_logitQnA1,
                                unadjusted_logitQnA0,
                                expanded_baseline_covars,
                                treatment,
                                scaled_outcome,
                                cvplan=cvplan)




    Estimate
          Estimate Std. Error Lower 95% CL Upper 95% CL
    ATE  0.0275032  0.0115604   0.00484526    0.0501612





    flucinfo(unadjusted_ctmle_est)




    Fluctuation info:
    Covariates added in 10 steps to 3 fluctuations.
    Fluc 1 covars added: [1,8,10,2,11]
    Fluc 2 covars added: [6,4,9,7]
    Fluc 3 covars added: [5]




So here we see three fluctuations, with a total of 10 covariates including the intercept being used to estimate $g_0$ out of 11 possible.  The estimate is nearly the same as the previous one, with a slightly larger standard error. 