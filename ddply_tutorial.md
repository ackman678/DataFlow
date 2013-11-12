Date: 2013-11-12 09:48:21   
Author: James B. Ackman  
Tags: programming, data analysis, R, statistics  

# Data summarization using ddply

A simple data.frame:

	actvFraction <- c(0.2838, 0.31114, 0.14775, 0.058811, 0.104, 0.079014, 0.015483)
	motorState <- c("quiet","quiet","quiet","active","active","active","active")
	df <-  data.frame(actvFraction, motorState)


Here is what the resulting data.frame looks like:

	> df
	  actvFraction motorState
	1     0.283800      quiet
	2     0.311140      quiet
	3     0.147750      quiet
	4     0.058811     active
	5     0.104000     active
	6     0.079014     active
	7     0.015483     active

## Base R summary functions

Summary from base package gives some useful info:

	> summary(df)
	  actvFraction      motorState
	 Min.   :0.01548   active:4   
	 1st Qu.:0.06891   quiet :3   
	 Median :0.10400              
	 Mean   :0.14286              
	 3rd Qu.:0.21577              
	 Max.   :0.31114 

Aggregate from base package. This one is works great, but is cumbersome to use on the fly for complex summaries unless you wrap it up inside your own custom function (like printSummary()):

	> aggregate(actvFraction ~ motorState,df,sum)
		  motorState actvFraction
	1     active     0.257308
	2      quiet     0.742690
	
## Use ddply from the plyr package instead

Usage:

	 ddply(.data, .variables, .fun = NULL, ...)
	 
Inputs:

* .data = data.frame to be processed
* .variables = variables to split data.frame by. Thus multiple factors can be based to this using .(x,y) or c("x","y") methods.
	* using as.quoted() variables
		* .() is the same as as.quoted() and are functions in the pylr package
	* a formula
	* a character vector
* .fun = function to apply to each piece
	* summarize() from the pylr package is a useful summary function that creates new dataframes and is often used in conjuction with ddply
* ... other arguments passed onto .fun

Here is the function applied to this data with sum, mean, standard deviation, N observations, and standard error calculated:  

	require(pylr)

	ddply(df, c("motorState"), summarize,
	sum = round(sum(actvFraction),2),
	mean = round(mean(actvFraction), 2),
	sd = round(sd(actvFraction), 2), 
	N = length(actvFraction), 
	se = sd/sqrt(N))

Resulting table:

	  motorState  sum mean   sd N         se
	1     active 0.26 0.06 0.04 4 0.02000000
	2      quiet 0.74 0.25 0.09 3 0.05196152


We can use an R formula of the descriptors, without the measured variable:

	myFormula <- ~ motorState
	ddply(df, myFormula, summarize,
	sum = round(sum(actvFraction),2),
	N = length(actvFraction))

The following one uses sum(!is.na()) instead of length, since length doesn't have an na.rm() option, together with other commonly desired summary outputs:

	ddply(df, c("motorState"), summarize,
	sum = round(sum(actvFraction),2),
	mean = round(mean(actvFraction),2),
	std = round(sd(actvFraction),2), 
	N = sum(!is.na(actvFraction)), 
	sem = std/sqrt(N),
	CI95 = qnorm(0.975)*sem,
	median = round(median(actvFraction),2),
	medSD = round(mad(actvFraction),2),
	seMed = 1.25*sem)

The resulting table: 

	  motorState  sum mean  std N        sem       CI95 median medSD      seMed
	1     active 0.26 0.06 0.04 4 0.02000000 0.03919928   0.07  0.03 0.02500000
	2      quiet 0.74 0.25 0.09 3 0.05196152 0.10184272   0.28  0.04 0.06495191


## Generic function and formula interface example

Assign your measured and group dataframe column names to new variables:

	measVar = "actvFraction"
	groupVars = c("motorState")

Make a formula of these variables to use for statistical testing: 

	myFormula <- as.formula(paste(measVar, paste(groupVars, collapse=" + "), sep=" ~ "))
	t.test(myFormula,df)
	wilcox.test(myFormula,df)

Make a generic R summary function based on `ddply` that can be used for any combination of measured observations and grouping factors:

	printSummary <- function(data=NULL, measVar, groupVars=NULL, na.rm=FALSE,
	                      conf.interval=.95, .drop=TRUE) {
		require(plyr)
	
		#Handle NAs
		len <- function (x, na.rm=FALSE) {
		    if (na.rm) sum(!is.na(x))
		    else       length(x)
		}
	
		#Main summary
		data2 <- ddply(data, groupVars, .drop=.drop,
		      .fun = function(xx, col) {
		        c(N    = len(xx[[col]], na.rm=na.rm),
		          sum = sum   (xx[[col]], na.rm=na.rm),
		          mean = mean   (xx[[col]], na.rm=na.rm),
		          sd   = sd     (xx[[col]], na.rm=na.rm)
		        )
		      },
		      measVar
		    )
	
		#Standard error of the mean
		data2$se <- data2$sd / sqrt(data2$N)
	
		#Confidence interval of 95%
		data2$CI95 = qnorm(0.975)*data2$se
	
		return(data2)
	}

*More examples and methods at [R cookbook](http://www.cookbook-r.com/Manipulating_data/Summarizing_data/).*

Call this function: 

	printSummary(df, measVar,groupVars)

Returned table of results:

	  motorState N      sum      mean         sd         se       CI95
	1     active 4 0.257308 0.0643270 0.03744248 0.01872124 0.03669295
	2      quiet 3 0.742690 0.2475633 0.08751511 0.05052687 0.09903085
