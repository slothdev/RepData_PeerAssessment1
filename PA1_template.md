# Reproducible Research: Peer Assessment 1
Edwin Seah  
04 March 2015  



## Loading and preprocessing the data  

1. Load the data (i.e. `read.csv()`)

2. Process/transform the data (if necessary) into a format suitable for your analysis

+ We unzip the raw zip file containing activity data into a specified data 
directory within our working directory, and load it using `read.csv()` into
a data frame called **df**. Integer is also specified as the type for *interval*
to simplify scaling and sorting later. We could also transform our intervals 
into *Date* or *Time* objects but for this analysis since they are 288 5-minute
intervals each day we can keep the original sequential integer form for the
intervals.


```r
unzip("activity.zip", exdir="data")
df <- read.csv("data/activity.csv", 
               header=TRUE, 
               colClasses=c("integer", "Date", "integer")
               )
```


## What is mean total number of steps taken per day?  

1. Make a histogram of the total number of steps taken each day

+ We make the histogram using a new data frame that will exclude the NA values. 
To create this new data frame (called **mts**), we use the **dplyr** package to
transform the raw data frame **df**:

    + Group by *date*
    + Add a column called *totalSteps* which is simply the sum of all steps in a day
    + Remove *interval* and *steps* columns so that we can group by 
    *date* and call `unique()` to isolate only unique rows
    + Results in new data frame **mts** comprising 2 columns, *date* and *totalSteps*

+ Plotting the histogram and specifying `breaks=10` allows for a reasonable number
of 20 bins in our histogram, such that we can better observe the estimated 
distribution.  


```r
library(dplyr)
# Create the NA-free data frame
mts <- df[complete.cases(df),] %>% 
    group_by(date) %>% 
    mutate(totalSteps=sum(steps), interval=NULL, steps=NULL) %>%
    unique()
# Plot the histogram
hist(mts$totalSteps, 
     breaks=20, 
     xlim=c(0,25000), 
     ylim=c(0,20),
     xlab="Total Steps", 
     main="Histogram of Total Number of Steps each Day")
```

![](figures/histogram_totalsteps_estimate-1.png) 

2. Calculate and report the **mean** and **median** total number of steps
taken per day

+ We simply call `mean()` and `median()` on the the newly-created data frame 
**mts**, the **mean** is **1.0766189\times 10^{4}** and the **median** is 
**10765**.  


```r
mean(mts$totalSteps)
```

```
## [1] 10766.19
```

```r
median(mts$totalSteps)
```

```
## [1] 10765
```


## What is the average daily activity pattern?

1. Make a time series plot (i.e. `type = "l"`) of the 5-minute interval (x-axis) 
and the average number of steps taken, averaged across all days (y-axis)

+ In order to make this plot, we first need to build a data frame for it, which
we will call **avgDaily**.
This can be neatly done with **dplyr** again, by piping only the non-NA
observations via `complete.cases()`, then calling `summarise_each()` on the 
grouped intervals to obtain the means across all days for every interval.
Then the plot can be made with `xyplot()` from the **lattice** plotting system.


```r
# Derive a data frame that excludes all non-NA values
avgDaily <- df[complete.cases(df),] %>% 
    group_by(interval) %>% 
    summarise_each(funs(mean), steps)
# PLot using lattice
library(lattice)
xyplot(steps ~ interval, 
       avgDaily, 
       type="l", 
       xlab="Interval",
       ylab="Average Number of Steps",
       main="Average Daily Activity Pattern of Steps per 5-min interval")
```

![](figures/plot_activity_daily-1.png) 

2. Which 5-minute interval, on average across all the days in the dataset, contains the maximum number of steps?

+ We use `select()` (from **dplyr**) on the **avgDaily** data frame to get the 
whole list of averaged number of steps per 5-minute interval, then use `max()` 
to find the specific 5-minute interval which has the maximum number of steps, 
which is  **835**.


```r
avgDaily[avgDaily$steps==max(avgDaily$steps),]$interval
```

```
## [1] 835
```


## Imputing missing values

1. Calculate and report the total number of missing values in the dataset (i.e. the total number of rows with NAs)  

+ We use `!complete.cases()` on the original data frame **df** (giving us 
all rows with NAs) and put it in a new data frame called **missing**. 
Using `nrow()` on **missing** to count the number of rows, gives us 
**2304** rows.


```r
missing <- df[!complete.cases(df),]
nrow(missing)
```

```
## [1] 2304
```

2. Devise a strategy for filling in all of the missing values in the dataset. The strategy does not need to be sophisticated. For example, you could use the mean/median for that day, or the mean for that 5-minute interval, etc.  

+ We observe that there are **2304** NA rows from the
original dataset, and **8** days when they do occur.
Dividing the NA rows by the number of days with NA rows gives us exactly 
**288**, which is the exact number of
5-minute intervals per day. Since we already have the mean across all days for 
every 5-minute interval in the data frame **avgDaily**, we can re-use them to fill
in the missing data.

3. Create a new dataset that is equal to the original dataset but with the missing data filled in.  

+ Using `merge()` we perform a left join by *interval* on the data frames **df** 
and **avgDaily** which we place in a new data frame called **filled**. 
There are now *steps.x* and *steps.y* in **filled** which can then be passed 
through `mutate()` to fill in the missing values where there are NAs.

```r
# Using mean for each 5-minute interval from avgDaily to fill in missing values
filled <- merge(df, avgDaily, by="interval", type="left", match="first")
filled <- filled  %>% 
    mutate(steps=(ifelse(is.na(steps.x), 
                         steps.y, 
                         steps.x)
                  ), 
           steps.x=NULL, 
           steps.y=NULL)
```

4. Make a histogram of the total number of steps taken each day and calculate and report the mean and median total number of steps taken per day. Do these values differ from the estimates from the first part of the assignment? What is the impact of imputing missing data on the estimates of the total daily number of steps?

+ Re-using the code from our first histogram to create the data frame 
**mtsImputed**, we end up with a histogram of the dataset with imputed total 
daily number of step values that shows a dramatically higher frequency for the
step range 10000-11000, presumably from the 8 missing days filled in.

+ Imputation of the missing values did not change the mean from its estimate, 
but does raise the imputed median slightly to match the mean.


```r
# Make mtsImputed re-using code from making mts
mtsImputed <- filled %>% 
    group_by(date) %>% 
    mutate(totalSteps=sum(steps), interval=NULL, steps=NULL) %>%
    unique()
# Make the histogram
hist(mtsImputed$totalSteps, 
     breaks=20, 
     xlim=c(0,25000), 
     ylim=c(0,20),
     xlab="Total Steps", 
     main="Histogram of Total Number of Steps each Day")
```

![](figures/histogram_totalsteps_imputed-1.png) 

**Mean and Median from Estimated set:**

```r
mean(mts$totalSteps)
```

```
## [1] 10766.19
```

```r
median(mts$totalSteps)
```

```
## [1] 10765
```

**Mean and Median from Imputed set:**

```r
mean(mtsImputed$totalSteps)
```

```
## [1] 10766.19
```

```r
median(mtsImputed$totalSteps)
```

```
## [1] 10766.19
```


## Are there differences in activity patterns between weekdays and weekends?  

1. Create a new factor variable in the dataset with two levels -- "weekday" and "weekend" indicating whether a given date is a weekday or weekend day.  

+ We mutate the *filled* data frame to add a new factor variable called 
*daytype*, which is populated as "weekend" if `weekdays(date)` is a `"Saturday"` 
or a `"Sunday"`, and "weekday" otherwise. 

```r
filled <- 
    filled %>% 
    mutate ( daytype = as.factor (
        ifelse ( 
            weekdays(date)=="Saturday" | weekdays(date)=="Sunday", 
            "weekend", 
            "weekday"))
        )
```

2. Make a panel plot containing a time series plot (i.e. `type = "l"`) of the 5-minute interval (x-axis) and the average number of steps taken, averaged across all weekday days or weekend days (y-axis). The plot should look something like the following, which was created using simulated data:

+ We use the same method previously used in plotting the average daily activity 
pattern from the estimated data, but additionally grouping by *daytype*.
Our panel plot is then created using `xyplot()` from the **lattice** package,
split into two panels by our new factor variable *daytype*.


```r
# Munge the filled data frame to get average daily pattern by daytype and interval
filled <- filled %>%
    group_by(daytype, interval) %>%
    summarise_each(funs(mean), steps)
# Plot a panel with 1 col and 2 rows using xyplot
xyplot(steps ~ interval | daytype, filled, type="l", layout=c(1,2))
```

![](figures/plot_activity_weekdays-1.png) 