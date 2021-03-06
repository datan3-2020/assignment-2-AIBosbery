Data analysis assignment 2
================
Andrew Bosbery
06/02/2020

In this assignment you will work with relational data, i.e. data coming
from different data tables that you can combine using keys. Please read
ch.13 from R for Data Science before completing this assignment –
<https://r4ds.had.co.nz/relational-data.html>.

## Read data

We will work with three different tables: household roster from wave 8
(*h\_egoalt*), stable characteristics of individuals (*xwavedat*), and
household data from wave 8 (*h\_hhresp*).

``` r
library(tidyverse)
```

    ## Warning: package 'tidyverse' was built under R version 3.5.3

    ## Warning: package 'ggplot2' was built under R version 3.5.3

    ## Warning: package 'tibble' was built under R version 3.5.3

    ## Warning: package 'tidyr' was built under R version 3.5.3

    ## Warning: package 'readr' was built under R version 3.5.3

    ## Warning: package 'purrr' was built under R version 3.5.3

    ## Warning: package 'dplyr' was built under R version 3.5.3

    ## Warning: package 'stringr' was built under R version 3.5.3

    ## Warning: package 'forcats' was built under R version 3.5.3

``` r
# You need to complete the paths to these files on your computer.

Egoalt8 <- read_tsv("C:/Users/Andrew/Documents/University/Year 2/Term 2/Data Analysis 3/Project/data/UKDA-6614-tab/tab/ukhls_w8/h_egoalt.tab")
Stable <- read_tsv("C:/Users/Andrew/Documents/University/Year 2/Term 2/Data Analysis 3/Project/data/UKDA-6614-tab/tab/ukhls_wx/xwavedat.tab")
Hh8 <- read_tsv("C:/Users/Andrew/Documents/University/Year 2/Term 2/Data Analysis 3/Project/data/UKDA-6614-tab/tab/ukhls_w8/h_hhresp.tab")
```

## Filter household roster data (10 points)

The **egoalt8** data table contains data on the kin and other
relationships between people in the same household. In each row in this
table you will have a pair of individuals in the same household: ego
(identified by *pidp*) and alter (identified by *apidp*).
*h\_relationship\_dv* shows the type of relationship between ego and
alter. You can check the codes in the Understanding Society codebooks
here –
<https://www.understandingsociety.ac.uk/documentation/mainstage/dataset-documentation>.

First we want to select only pairs of individuals who are husbands and
wives or cohabiting partners (codes 1 and 2). For convenience, we also
want to keep only the variables *pidp*, *apidp*, *h\_hidp* (household
identifier), *h\_relationship\_dv*, *h\_esex* (ego’s sex), and *h\_asex*
(alter’s sex).

``` r
Partners8 <- Egoalt8 %>%
        filter(h_relationship_dv == 1 | h_relationship_dv == 2) %>%
        select(pidp, apidp, h_hidp, h_relationship_dv, h_sex, h_asex)
```

Each couple now appears in the data twice: 1) with one partner as ego
and the other as alter, 2) the other way round. Now we will only focus
on heterosexual couples, and keep one observation per couple with women
as egos and men as their alters.

``` r
Hetero8 <- Partners8 %>%
        # filter out same-sex couples
        filter(h_sex != h_asex) %>%
        # keep only one observation per couple with women as egos
        filter(h_sex == 2 & h_asex == 1)  #h_asex == 1 unnecessary but a good safeguard
```

## Recode data on ethnicity (10 points)

In this assignment we will explore ethnic endogamy, i.e. marriages and
partnerships within the same ethnic group. First, let us a create a
version of the table with stable individual characteristics with two
variables only: *pidp* and *racel\_dv* (ethnicity).

``` r
Stable2 <- Stable %>%
        select(pidp, racel_dv)
```

Let’s code missing values on ethnicity (-9) as NA.

``` r
Stable2 <- Stable2 %>%
        mutate(racel_dv = recode(racel_dv, `-9` = NA_real_))
```

Now let us recode the variable on ethnicity into a new binary variable
with the following values: “White” (codes 1 to 4) and “non-White” (all
other codes).

``` r
Stable2 <- Stable2 %>%
        mutate(race = as.character(case_when(
                between(racel_dv, 1, 4) ~ "white",
                racel_dv > 4 ~ "non-white")))
                
        #NAs introduced by coercion
```

## Join data (30 points)

Now we want to join data from the household roster (*Hetero8*) and the
data table with ethnicity (*Stable2*). First let us merge in the data on
ego’s ethnicity. We want to keep all the observations we have in
*Hetero8*, but we don’t want to add any other individuals from
*Stable2*.

``` r
JoinedEthn <- Hetero8 %>%
        left_join(Stable2, by = "pidp")
```

Let us rename the variables for ethnicity to clearly indicate that they
refer to egos.

``` r
JoinedEthn <- JoinedEthn %>%
        rename(egoRacel_dv = racel_dv) %>%
        rename(egoRace = race)
```

Now let us merge in the data on alter’s ethnicity. Note that in this
case the key variables have different names in two data tables; please
refer to the documentation for your join function (or the relevant
section from R for Data Science) to check the solution for this problem.

``` r
JoinedEthn <- JoinedEthn %>%
        left_join(Stable2, by = c("apidp" = "pidp"))
```

Renaming the variables for alters.

``` r
JoinedEthn <- JoinedEthn %>%
        rename(alterRacel_dv = racel_dv) %>%
        rename(alterRace = race)
```

## Explore probabilities of racial endogamy (20 points)

Let us start by looking at the joint distribution of race (White
vs. non-White) of both partners.

``` r
TableRace <- JoinedEthn %>%
        # filter out observations with missing data
        filter(alterRace != is.na(alterRace) & egoRace != is.na(egoRace)) %>%
        count(egoRace, alterRace)
TableRace
```

    ## # A tibble: 4 x 3
    ##   egoRace   alterRace     n
    ##   <chr>     <chr>     <int>
    ## 1 non-white non-white  1790
    ## 2 non-white white       326
    ## 3 white     non-white   266
    ## 4 white     white      9694

Now calculate the following probabilities: 1) for a White woman to have
a White partner, 2) for a White woman to have a non-White partner, 3)
for a non-White woman to have a White partner, 4) for a non-White woman
to have a non-White partner.

Of course, you can simply calculate these numbers manually. However, the
code will not be reproducible: if the data change the code will need to
be changed, too. Your task is to write reproducible code producing a
table with the required four probabilities.

``` r
TableRace %>%
        # group by ego's race to calculate sums
        group_by(egoRace) %>%
        # create a new variable with the total number of women by race
        mutate(Female_Race_Total = sum(n)
               ) %>%
        # create a new variable with the required probabilities 
        mutate(Alter_Race_Prob = (n/Female_Race_Total) * 100)
```

    ## # A tibble: 4 x 5
    ## # Groups:   egoRace [2]
    ##   egoRace   alterRace     n Female_Race_Total Alter_Race_Prob
    ##   <chr>     <chr>     <int>             <int>           <dbl>
    ## 1 non-white non-white  1790              2116           84.6 
    ## 2 non-white white       326              2116           15.4 
    ## 3 white     non-white   266              9960            2.67
    ## 4 white     white      9694              9960           97.3

## Join with household data and calculate mean and median number of children by ethnic group (30 points)

1)  Join the individual-level file with the household-level data from
    wave 8 (specifically, we want the variable for the number of
    children in the household).
2)  Select only couples that are ethnically endogamous (i.e. partners
    come from the same ethnic group) for the following groups: White
    British, Indian, and Pakistani.
3)  Produce a table showing the mean and median number of children in
    these households by ethnic group (make sure the table has meaningful
    labels for ethnic groups, not just numerical codes).
4)  Write a short interpretation of your results. What could affect your
    findings?

<!-- end list -->

``` r
NKids_Households <- Hh8 %>% 
        select(h_hidp, h_nkids_dv)
#A table with only the identifier and the number of kids per household

JoinedEthn <- JoinedEthn %>%
        left_join(NKids_Households, by = "h_hidp")

#adding the number of kids per household to the couples table
```

``` r
##Choosing only endogamous couples

Same_Ethnic <- JoinedEthn %>%
        filter(egoRace == alterRace) %>%
        filter(egoRacel_dv == 1 | egoRacel_dv == 9 | egoRacel_dv == 10) #%>%
        #filter(h_nkids_dv.x != is.na(h_nkids_dv.x))
```

``` r
Couple_Table <- Same_Ethnic %>%
        count(egoRacel_dv, h_nkids_dv)
        

        
Couple_Table
```

    ## # A tibble: 26 x 3
    ##    egoRacel_dv h_nkids_dv     n
    ##          <dbl>      <dbl> <int>
    ##  1           1          0  5965
    ##  2           1          1  1204
    ##  3           1          2  1266
    ##  4           1          3   311
    ##  5           1          4    64
    ##  6           1          5    16
    ##  7           1          6     4
    ##  8           1          7     1
    ##  9           1         NA   192
    ## 10           9          0   242
    ## # ... with 16 more rows

``` r
Couple_Table %>%
      mutate(weightedkids = n * h_nkids_dv) %>%
      mutate(totalcouplesrace = case_when(
         egoRacel_dv == 1 ~ sum(n[egoRacel_dv == 1]),
         egoRacel_dv == 9 ~ sum(n[egoRacel_dv == 9]),
         egoRacel_dv == 10 ~sum(n[egoRacel_dv == 10])
      )) %>%
       mutate(avergaekids = case_when( 
        egoRacel_dv == 1 ~  sum(weightedkids[egoRacel_dv == 1], na.rm = TRUE)/totalcouplesrace,
        egoRacel_dv == 9 ~  sum(weightedkids[egoRacel_dv == 9], na.rm = TRUE)/totalcouplesrace,
        egoRacel_dv == 10 ~  sum(weightedkids[egoRacel_dv == 10], na.rm = TRUE)/totalcouplesrace
        
        
        
        )
        

       )
```

    ## # A tibble: 26 x 6
    ##    egoRacel_dv h_nkids_dv     n weightedkids totalcouplesrace avergaekids
    ##          <dbl>      <dbl> <int>        <dbl>            <int>       <dbl>
    ##  1           1          0  5965            0             9023       0.558
    ##  2           1          1  1204         1204             9023       0.558
    ##  3           1          2  1266         2532             9023       0.558
    ##  4           1          3   311          933             9023       0.558
    ##  5           1          4    64          256             9023       0.558
    ##  6           1          5    16           80             9023       0.558
    ##  7           1          6     4           24             9023       0.558
    ##  8           1          7     1            7             9023       0.558
    ##  9           1         NA   192           NA             9023       0.558
    ## 10           9          0   242            0              534       0.938
    ## # ... with 16 more rows

On average white couples have 0.56 childern per household, Indian
couples have 0.93 children per household and Pakistani couples have 1.70
children per household. This suggests that white British couples have
less children than couples from these two ethnicities and Pakistani
couples the most. Socio-economic factors could be a reason for this.
This could be that many of the Pakistani and Indian couples have a less
secure socio-economic position, they earn less money and do not have
much in terms of savings. This could mean that these couples have more
children to support them when they retire unlike the white couples who
may have a relatively better off income and so do not need as many
children to support them in the future.

Theses findings may be affected by many factors. One of these the is not
controlling for age and professional status. This table does not account
for the ages of participants or their professional status therefore it
is possible this may be a confounding factor. For example, if an
unrepresentative amount of the white British couples are young
professionals and Indian and Pakistani couples and unrepresentively
older with only working than this could explain why white British
couples have on average less children than their Indian and Pakistani
counterparts. This is because two young professionals may have less time
to run a family than an older couple one of whom will be free to look
after the kids. If this was accounted for there may have been a smaller
difference in the average amount of children at different ages or
professional status.
