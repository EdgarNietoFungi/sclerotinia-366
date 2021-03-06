---
title: "R Notebook"
output: 
  html_notebook:
    toc: true
---

This document serves to compare the data presented in Sydney's original XLSX 
file and the one generated by Sajeewa. He cleaned it up by inserting "unk" for
missing values. Because there was no file trail for this other than his own
comment, I will compare these two.





```r
library('tidyverse')
```

```
## ── Attaching packages ────────────────────────────────── tidyverse 1.2.1 ──
```

```
## ✔ ggplot2 2.2.1     ✔ purrr   0.2.4
## ✔ tibble  1.3.4     ✔ dplyr   0.7.4
## ✔ tidyr   0.7.2     ✔ stringr 1.2.0
## ✔ readr   1.1.1     ✔ forcats 0.2.0
```

```
## ── Conflicts ───────────────────────────────────── tidyverse_conflicts() ──
## ✖ dplyr::filter() masks stats::filter()
## ✖ dplyr::lag()    masks stats::lag()
```

```r
library('readxl')
library('assertr')
library('poppr')
```

```
## Loading required package: adegenet
```

```
## Loading required package: ade4
```

```
## 
##    /// adegenet 2.1.0 is loaded ////////////
## 
##    > overview: '?adegenet'
##    > tutorials/doc/questions: 'adegenetWeb()' 
##    > bug reports/feature requests: adegenetIssues()
```

```
## This is poppr version 2.5.0. To get started, type package?poppr
## OMP parallel support: available
```


## Assertations

Of course, to make sure the data are cromulent, we want to ensure they live up to
our standards of what has been reported. For this, we will use the *assertr*
package, which lets us check our data in various ways. I'm taking the data
presented in the paper thus far to create the assertions.



```r
size_ranges <- 
"locus\trange\tn
5-2(F)	318-324	4
6-2(F)	483-495	3
7-2(F)	158-174	7
8-3(H)	244-270	7
9-2(F)	360-382	9
12-2(H)	214-222	5
17-3(H)	342-363	7
20-3(F)	280-282	2
55-4(F)	153-216	10
110-4(H)	370-386	5
114-4(H)	339-416	10
"
sr <- read_tsv(size_ranges) %>%
  separate(range, c("lower", "upper"), sep = "-") # split range into lower and upper bounds
sr
```

```
## # A tibble: 11 x 4
##       locus lower upper     n
##  *    <chr> <chr> <chr> <int>
##  1   5-2(F)   318   324     4
##  2   6-2(F)   483   495     3
##  3   7-2(F)   158   174     7
##  4   8-3(H)   244   270     7
##  5   9-2(F)   360   382     9
##  6  12-2(H)   214   222     5
##  7  17-3(H)   342   363     7
##  8  20-3(F)   280   282     2
##  9  55-4(F)   153   216    10
## 10 110-4(H)   370   386     5
## 11 114-4(H)   339   416    10
```

```r
# Making the assertation string.
# To do this, we are using `sprintf()` to print the assert statement:
#   assert(within_bounds(lower, upper), `locus`)
# which means: "assert that the alleles (data) within this locus (variable) are
# within the defined range". 
# 
# The `apply()` function takes a matrix or data frame and applies a given function over each
# row (MARGIN = 1) or column (MARGIN = 2). In this case, we are specifying rows.
# 
# We then collapse all these statements with the pipe operator: %>%
# Then we add the name of the operation to the beginning
assrt_string <- apply(sr, MARGIN = 1, function(x){
    sprintf("  assert(within_bounds(%s, %s), `%s`)", x[2], x[3], x[1]) 
  }) %>% 
  paste(collapse = " %>%\n") %>%
  paste("assert_alleles_within_bounds <- .", ., sep = " %>%\n")
#
# now we can see what this statement looks like
cat(assrt_string)
```

```
## assert_alleles_within_bounds <- . %>%
##   assert(within_bounds(318, 324), `5-2(F)`) %>%
##   assert(within_bounds(483, 495), `6-2(F)`) %>%
##   assert(within_bounds(158, 174), `7-2(F)`) %>%
##   assert(within_bounds(244, 270), `8-3(H)`) %>%
##   assert(within_bounds(360, 382), `9-2(F)`) %>%
##   assert(within_bounds(214, 222), `12-2(H)`) %>%
##   assert(within_bounds(342, 363), `17-3(H)`) %>%
##   assert(within_bounds(280, 282), `20-3(F)`) %>%
##   assert(within_bounds(153, 216), `55-4(F)`) %>%
##   assert(within_bounds(370, 386), `110-4(H)`) %>%
##   assert(within_bounds(339, 416), `114-4(H)`)
```

```r
# here, we evaluate it, making it available for us later on.
eval(parse(text = assrt_string)) 

YEARS <- c("2003", "2004", "2005", "2007", "2008", "2009", "2010", "2012")

STATE <- c("AU", "TS", "FR", "BL", "MX", "NE", "NY", "MN", "MI", "OR", 
           "WA", "CO", "WI", "ID", "CA", "ND")

HOST <- c("Merlot", "Pinto", "redkid", "Beryl", "Bunsi", "37", "38", 
          "11A", "cornell", "G122", "Orion", "PO7104", "PO7863", "WM31", 
          "GH", "BO7104", "Black", "Vista", "SR06233", "BL", "Fuji", "unk", 
          "Zorro", "PO7883", "Emerson", "Weihing", "Yellow")

check_data_cromulence <- . %>%
  chain_start() %>%
  verify(nrow(.) == 366) %>%          # 366 isolates
  verify(has_all_names(sr$locus)) %>% # Has all loci
  assert_alleles_within_bounds %>%    # each locus has alleles within bounds
  assert(in_set(YEARS), Year) %>%     # has all specified years
  assert(in_set(STATE), State) %>%    # has all specified States (incl. countries)
  assert(in_set(HOST), Host) %>%      # has all specified Hosts (incl. countries)
  chain_end(error_fun = warn_report)
```


## Reading in Sydney's data

Sydney's data is stored in excel format, so we have to use readxl to parse it.
First, we want to know what sheets exist.


```r
excel_sheets(file.path(PROJHOME, "data/raw/2017-02-16_copy-of-binned-genotypes_SE.xlsx"))
```

```
## [1] "original"      "edited"        "GenAlex"       "replen est"   
## [5] "GenAlexBinned"
```

We will be using the GenAlexBinned sheet. From my talks with Sydney, this sheet
contains the SSR data binned into the expected allelels. Note, this sequence of
data input and cleaning was done iteratively and what you are seeing is the 
final result.

There are always quirks with the data when it's in excel. Often, there are too
many rows, so we have to remove them with `slice()`.

> 2017-06-11: This has been fixed in readxl version 1.0

Quirks specific to thse data:

 - GenAlEx format has an extra header region that should be ignored
 - The loci are formatted like so: `110-4(F)`, where the `(F)` part is not 
   informative
 - The header for the strata has extra information
 

 

```r
syd <- read_excel(file.path(PROJHOME, "data/raw/2017-02-16_copy-of-binned-genotypes_SE.xlsx"),
                sheet = "GenAlexBinned", skip = 1) %>%
  select(-1) %>%                # removing first column, which is empty
  gather(locus, allele, -1) %>% # gather all loci into tidy columns
  mutate(locus = trimws(locus)) %>% # remove whitespace in locus names
  mutate(allele = as.integer(allele)) %>% # force alleles to integers
  spread(locus, allele) %>%     # spread data out with individual loci in columns
  separate(iso_st_mcg_org_loc_yr_hst_cult_rep, 
           c("Isolate", "Severity", "MCG", "State", "Source", "Year", "Host"), 
           sep = "_") %>%
  mutate_if(is.character, trimws) %>% # Trim whitespace forom all character columns
  mutate(Severity = as.numeric(Severity)) %>%
  mutate(Source = ifelse(Source == "", "unk", Source)) %>%
  # slice(-n()) %>% # remove last row
  arrange(Isolate) %>%
  check_data_cromulence
```

```
## There is 1 error:
## 
## - Column 'Host' violates assertion 'in_set(HOST)' 3 times
##   index  value
## 1    39 ExRico
## 2   263 B07104
## 3   317 PO7683
```

```
## Warning: assertr encountered errors
```

```r
syd
```

```
## # A tibble: 366 x 18
##    Isolate Severity   MCG State Source  Year  Host `110-4(H)` `114-4(H)`
##      <chr>    <dbl> <chr> <chr>  <chr> <chr> <chr>      <int>      <int>
##  1     152      3.9     4    NE    unk  2003    GH        378        371
##  2     274      5.4    45    NE    unk  2003    GH        378        359
##  3     443      6.3     5    NY    unk  2003    GH        378        367
##  4     444      4.4     4    MN    wmn  2003  G122        378        371
##  5     445      4.7     4    MN    wmn  2003 Beryl        378        371
##  6     446      6.1     3    MI    wmn  2003 Beryl        386        359
##  7     447      5.5     5    MI    wmn  2003 Beryl        378        367
##  8     448      5.0     3    MI    wmn  2003 Beryl        386        359
##  9     449      5.2     3    MI    wmn  2003 Bunsi        386        367
## 10     450      5.3     5    MI    wmn  2003 Bunsi        378        367
## # ... with 356 more rows, and 9 more variables: `12-2(H)` <int>,
## #   `17-3(H)` <int>, `20-3(F)` <int>, `5-2(F)` <int>, `55-4(F)` <int>,
## #   `6-2(F)` <int>, `7-2(F)` <int>, `8-3(H)` <int>, `9-2(F)` <int>
```

## Reading in Sajeewa's data

These data can be read in via `reader::read_csv()`


```r
column_specification <- cols(
  Individual = col_character(),
  iso_st_mcg_org_loc_yr_hst = col_character(),
  `5-2(F)` = col_integer(),
  `6-2(F)` = col_integer(),
  `7-2(F)` = col_integer(),
  `8-3(H)` = col_integer(),
  `9-2(F)` = col_integer(),
  `12-2(H)` = col_integer(),
  `17-3(H)` = col_integer(),
  `20-3(F)` = col_integer(),
  `55-4(F)` = col_integer(),
  `110-4(H)` = col_integer(),
  `114-4(H)` = col_integer()
)
saj <- read_csv(file.path(PROJHOME, "data/raw/2017-02-16_binned-genotypes-genalex_SA.csv"), skip = 2, col_types = column_specification) %>%
  select(-1) %>%
  gather(locus, allele, -1) %>% # gather all loci into tidy columns
  mutate(locus = trimws(locus)) %>% # remove any whitespace in the locus names
  spread(locus, allele) %>%     # spread data out with individual loci in columns
  separate(iso_st_mcg_org_loc_yr_hst, # Note: this corresponds closer to the initial data
           c("Isolate", "Severity", "MCG", "State", "Source", "Year", "Host"), 
           sep = "_") %>%
  mutate_if(is.character, trimws) %>% # Trim whitespace forom all character columns
  mutate(Severity = as.numeric(Severity)) %>%
  arrange(Isolate) %>%
  check_data_cromulence
```

```
## There is 1 error:
## 
## - Column 'Host' violates assertion 'in_set(HOST)' 2 times
##   index  value
## 1   263 B07104
## 2   317 PO7683
```

```
## Warning: assertr encountered errors
```

```r
saj
```

```
## # A tibble: 366 x 18
##    Isolate Severity   MCG State Source  Year  Host `110-4(H)` `114-4(H)`
##      <chr>    <dbl> <chr> <chr>  <chr> <chr> <chr>      <int>      <int>
##  1     152      3.9     4    NE    unk  2003    GH        378        371
##  2     274      5.4    45    NE    unk  2003    GH        378        359
##  3     443      6.3     5    NY    unk  2003    GH        378        367
##  4     444      4.4     4    MN    wmn  2003  G122        378        371
##  5     445      4.7     4    MN    wmn  2003 Beryl        378        371
##  6     446      6.1     3    MI    wmn  2003 Beryl        386        359
##  7     447      5.5     5    MI    wmn  2003 Beryl        378        367
##  8     448      5.0     3    MI    wmn  2003 Beryl        386        359
##  9     449      5.2     3    MI    wmn  2003 Bunsi        386        367
## 10     450      5.3     5    MI    wmn  2003 Bunsi        378        367
## # ... with 356 more rows, and 9 more variables: `12-2(H)` <int>,
## #   `17-3(H)` <int>, `20-3(F)` <int>, `5-2(F)` <int>, `55-4(F)` <int>,
## #   `6-2(F)` <int>, `7-2(F)` <int>, `8-3(H)` <int>, `9-2(F)` <int>
```

## Comparison


We've sorted each data set by Isolate, so that means that we should expect them
to have the same data. The `setequal()` checks whether or not both data sets
have the same rows (in any order)


```r
dplyr::setequal(saj, syd)
```

```
## FALSE: Rows in x but not y: 366, 365, 364, 260, 363, 41, 40, 39, 38. Rows in y but not x: 366, 365, 364, 363, 260, 41, 40, 39, 38.
```

Okay, something's not cromulent here. We'll have to manaully inspect these:


```r
syddiff <- dplyr::setdiff(syd, saj)
sajdiff <- dplyr::setdiff(saj, syd)
the_difference <- bind_rows(syd = syddiff, saj = sajdiff, .id = "source") %>% arrange(Isolate)
head(the_difference, n = 20)
```

```
## # A tibble: 18 x 19
##    source Isolate Severity   MCG State Source  Year   Host `110-4(H)`
##     <chr>   <chr>    <dbl> <chr> <chr>  <chr> <chr>  <chr>      <int>
##  1    syd     496      5.1    83    TS    wmn  2004   G122        386
##  2    saj     496      5.1    83    AU    wmn  2004   G122        386
##  3    syd     499      5.6    85    TS    wmn  2004 ExRico        378
##  4    saj     499      5.6    85    AU    wmn  2004  Bunsi        378
##  5    syd     500      1.4    83    TS    wmn  2004  Beryl        374
##  6    saj     500      1.4    83    AU    wmn  2004  Beryl        374
##  7    syd     501      3.3    84    TS    wmn  2004  Beryl        374
##  8    saj     501      3.3    84    AU    wmn  2004  Beryl        374
##  9    saj     805      5.8    18    ND    stc  2010    unk        382
## 10    syd    805*      5.8    18    ND    stc  2010    unk        382
## 11    syd     966      4.5    33    BL   flds  2012    unk        386
## 12    saj     966      4.5    33    FR   flds  2012    unk        386
## 13    syd     967      5.8    34    BL   flds  2012    unk        386
## 14    saj     967      5.8    34    FR   flds  2012    unk        386
## 15    syd     968      4.2    34    BL   flds  2012    unk        370
## 16    saj     968      4.2    34    FR   flds  2012    unk        370
## 17    syd     970      5.2    35    BL   flds  2012    unk        382
## 18    saj     970      5.2    35    FR   flds  2012    unk        382
## # ... with 10 more variables: `114-4(H)` <int>, `12-2(H)` <int>,
## #   `17-3(H)` <int>, `20-3(F)` <int>, `5-2(F)` <int>, `55-4(F)` <int>,
## #   `6-2(F)` <int>, `7-2(F)` <int>, `8-3(H)` <int>, `9-2(F)` <int>
```


## Conclusion

Both data sets failed the cromulence check in a common manner:

```
- Column 'Host' violates assertion 'in_set(HOST)' 2 times
  index  value
1   263 B07104
2   317 PO7683
```

These are the values I grabbed from the manuscript that could possibly match:

 - `PO7863`
 - `PO7883`
 - `BO7104` (The difference is that this is the letter "O" and above is the number "0")

Sydney's data set had an extra discrepancy, but that's noted in the difference
between data sets below:

1. Isolate 805 is flagged for some reason in Sydney's data set
2. Tasmania (TS, Sydney) has been changed to Australia (AU, Sajeewa)
3. Belgium (BL, Sydney) has been changed to France (FR, Sajeewa)
4. Host for isolate 499 has been changed from ExRico (Sydney) to Bunsi (Sajeewa)

### Sydney's comments

We have no clue why 805 is flagged. The changes to the metadata are correct.
The change from BL to FR was because of confusion between the source of the 
collectors vs. source of isolate.

TS was grouped into AU because it was too small of a sample size to do anything.


From here on out, we will use Sajeewa's data for analysis.


## Saving data as genclone object

First we need to transform the data to something that is useable, aka a genclone
object. To avoid confustion with states, we are also transforming the two-letter
abbreviations for France, Mexico, and Australia to the full names. 


```r
dat <- saj %>% 
  select(-(Isolate:Host)) %>% 
  df2genind(ind.names = saj$Isolate, 
            strata = select(saj, Severity:Host), # Filtering out severity and Isolate
            ploidy = 1) %>%
  as.genclone()
nameStrata(dat)[3] <- "Region"
pops <- levels(strata(dat)$Region)
levels(strata(dat)$Region) <- case_when(pops == "FR" ~ "France", 
                                        pops == "MX" ~ "Mexico", 
                                        pops == "AU" ~ "Australia", 
                                        TRUE ~ pops)
dat
```

```
## 
## This is a genclone object
## -------------------------
## Genotype information:
## 
##    165 original multilocus genotypes 
##    366 haploid individuals
##     11 codominant loci
## 
## Population information:
## 
##      6 strata - Severity, MCG, Region, Source, Year, Host
##      0 populations defined.
```

```r
locNames(dat)
```

```
##  [1] "110-4(H)" "114-4(H)" "12-2(H)"  "17-3(H)"  "20-3(F)"  "5-2(F)"  
##  [7] "55-4(F)"  "6-2(F)"   "7-2(F)"   "8-3(H)"   "9-2(F)"
```

### Gathering Repeat Lengths

The repeat lengths were contained in the `A1_Copy of binned-genotypes_SE.xlsx`
spreadsheet in the "replen est" sheet. Because this sheet contained several
merged cells and other oddities, I copy+pasted the relevant data here and 
parsed it from the text.


```r
rl <- read_tsv("dinuc			compound			hexinuc			dinuc			dinuc			dinuc			dinuc			trinuc			dinuc			compound			compound			tetranuc			dinuc			tetranuc			tetranuc			tetranuc
(GT)8			[(GT)2GAT]3(GT)14GAT(GT)5[GAT(GT)4]3(GAT)3			(TTTTTC)2(TTTTTG)2(TTTTTC)			(GA)14			(CA)12			(CA)9(CT)9			(CA)9			(TTA)9			(GT)7GG(GT)5			(CA)6(CGCA)2(CAT)2			(CA)7(TACA)2			(TACA)10			(CT)12			(CATA)25			(TATG)9			(AGAT)14(AAGC)4
 5-2(F)	bins	count	 5-3(F)	bins	count	 6-2(F)	bins	count	 7-2(F)	bins	count	 8-3(H)	bins	count	 9-2(F)	bins	count	 12-2(H)	bins	count	 17-3(H)	bins	count	 20-3(F)	bins	count	 36-4(F)	bins	count	 50-4(F)	bins	count	 55-4(F)	bins	count	 92-4(F)	bins	count	 106-4(H)	bins	count	110-4(H)	bins	count	114-4(H)")
```

```
## Warning: Missing column names filled in: 'X2' [2], 'X3' [3], 'X5' [5],
## 'X6' [6], 'X8' [8], 'X9' [9], 'X11' [11], 'X12' [12], 'X14' [14],
## 'X15' [15], 'X17' [17], 'X18' [18], 'X20' [20], 'X21' [21], 'X23' [23],
## 'X24' [24], 'X26' [26], 'X27' [27], 'X29' [29], 'X30' [30], 'X32' [32],
## 'X33' [33], 'X35' [35], 'X36' [36], 'X38' [38], 'X39' [39], 'X41' [41],
## 'X42' [42], 'X44' [44], 'X45' [45]
```

```
## Warning: Duplicated column names deduplicated: 'dinuc' => 'dinuc_1' [10],
## 'dinuc' => 'dinuc_2' [13], 'dinuc' => 'dinuc_3' [16], 'dinuc' =>
## 'dinuc_4' [19], 'dinuc' => 'dinuc_5' [25], 'compound' => 'compound_1' [28],
## 'compound' => 'compound_2' [31], 'dinuc' => 'dinuc_6' [37], 'tetranuc'
## => 'tetranuc_1' [40], 'tetranuc' => 'tetranuc_2' [43], 'tetranuc' =>
## 'tetranuc_3' [46]
```

```r
x <- rl %>%
  select(-starts_with("X")) %>% # removing non-informative columns
  slice(-1) %>%                 # removing first row (repeat patterns)
  gather(replen, locus) %>%
  mutate(replen = strsplit(replen, "_") %>% map_chr(1)) %>% # fix duplicated column names
  mutate(replen = case_when(
    grepl("^di", .$replen)    ~ 2,
    grepl("^tri", .$replen)   ~ 3,
    grepl("^tetra", .$replen) ~ 4,
    grepl("^hex", .$replen)   ~ 6,
    TRUE                      ~ 0.5
  ))

x <- paste("c(", paste(paste0("`", x$locus, "`", " = ", x$replen), collapse = ",\n"), ")", sep = "\n")
cat(x)
```

```
## c(
## `5-2(F)` = 2,
## `5-3(F)` = 0.5,
## `6-2(F)` = 6,
## `7-2(F)` = 2,
## `8-3(H)` = 2,
## `9-2(F)` = 2,
## `12-2(H)` = 2,
## `17-3(H)` = 3,
## `20-3(F)` = 2,
## `36-4(F)` = 0.5,
## `50-4(F)` = 0.5,
## `55-4(F)` = 4,
## `92-4(F)` = 2,
## `106-4(H)` = 4,
## `110-4(H)` = 4,
## `114-4(H)` = 4
## )
```

```r
repeat_lengths <- eval(parse(text = x))
repeat_lengths <- ifelse(repeat_lengths < 1, 4, repeat_lengths)
repeat_lengths
```

```
##   5-2(F)   5-3(F)   6-2(F)   7-2(F)   8-3(H)   9-2(F)  12-2(H)  17-3(H) 
##        2        4        6        2        2        2        2        3 
##  20-3(F)  36-4(F)  50-4(F)  55-4(F)  92-4(F) 106-4(H) 110-4(H) 114-4(H) 
##        2        4        4        4        2        4        4        4
```

### Adding inconsistent loci

Only 11 loci were manually curated by Sydney and Sajeewa. I'm taking a look at
the other 5 loci because they may be informative. In order to do that, I'm going
to have to clean them up by estimating the the true allele size.


```r
ex <- readxl::read_excel(file.path(PROJHOME, "data/raw/2017-02-16_copy-of-binned-genotypes_SE.xlsx"), sheet = "GenAlex", skip = 1) %>%
  select(-1) %>%                # removing first column, which is empty
  gather(locus, allele, -1) %>% # gather all loci into tidy columns
  mutate(locus = trimws(locus)) %>% # remove (F) designator
  mutate(allele = as.integer(allele)) %>% # force alleles to integers
  spread(locus, allele)

readr::write_csv(ex, "data/raw_data.csv", col_names = TRUE)

ex <- ex[!names(ex) %in% locNames(dat)]

# Function to select an adjacent allele. It will select the
# next allele if the next allele is not missing and it's distance
# is one away and the previous allele for the same conditions.
# If none of the conditions are met, it will retain the allele.
cromulent_allele <- Vectorize(function(lower, allele, higher){
  if (!is.na(higher) && abs(allele - higher) == 1){
    out <- higher
  } else if (!is.na(lower) && abs(allele - lower) == 1){
    out <- lower
  } else {
    out <- allele
  }
  out
})
ex
```

```
## # A tibble: 366 x 6
##    iso_st_mcg_org_loc_yr_hst_cult_rep `106-4(H)` `36-4(F)` `5-3(F)`
##                                 <chr>      <int>     <int>    <int>
##  1              152_3.9_4_NE__2003_GH        580       415      328
##  2             274_5.4_45_NE__2003_GH        588       415      328
##  3              443_6.3_5_NY__2003_GH        567       415      308
##  4         444_4.4_4_MN_wmn_2003_G122        580       415      328
##  5        445_4.7_4_MN_wmn_2003_Beryl        580       415      328
##  6        446_6.1_3_MI_wmn_2003_Beryl        567       415      339
##  7        447_5.5_5_MI_wmn_2003_Beryl        567       415      308
##  8          448_5_3_MI_wmn_2003_Beryl        568       414      339
##  9        449_5.2_3_MI_wmn_2003_Bunsi        568       415      339
## 10        450_5.3_5_MI_wmn_2003_Bunsi        568       415      308
## # ... with 356 more rows, and 2 more variables: `50-4(F)` <int>,
## #   `92-4(F)` <int>
```

```r
exsummary <- ex %>% 
  gather(locus, allele, -1) %>% # tidy the data
  group_by(locus, allele) %>%   
  summarize(n = n()) %>%        # summarize by count 
  ungroup() %>%
  group_by(locus) %>%           # group the loci, add the lower and upper alleles,
  mutate(lower = lag(allele), higher = lead(allele)) %>% # and then create new_alleles
  mutate(new_allele = ifelse(n < 3, cromulent_allele(lower, allele, higher), allele)) %>%
  select(locus, new_allele, allele)
exsummary
```

```
## # A tibble: 71 x 3
## # Groups:   locus [5]
##       locus new_allele allele
##       <chr>      <int>  <int>
##  1 106-4(H)        502    501
##  2 106-4(H)        502    502
##  3 106-4(H)        502    503
##  4 106-4(H)        511    511
##  5 106-4(H)        532    532
##  6 106-4(H)        533    533
##  7 106-4(H)        541    540
##  8 106-4(H)        541    541
##  9 106-4(H)        541    542
## 10 106-4(H)        546    546
## # ... with 61 more rows
```

Now that we have our data set filtered (to a degree), we can merge the data with
`dat` that we defined above.


```r
corrected_loci <- ex %>% gather(locus, allele, -1) %>%
  left_join(exsummary, by = c("locus", "allele")) %>%
  mutate(allele = new_allele) %>%
  select(-new_allele) %>%
  spread(locus, allele) %>%
  separate(iso_st_mcg_org_loc_yr_hst_cult_rep, # Note: this corresponds closer to the initial data
           c("Isolate", "Severity", "MCG", "State", "Source", "Year", "Host"), 
           sep = "_") %>%
  arrange(Isolate) %>%
  select(-(MCG:Host))
datdf <- genind2df(dat, usepop = FALSE) %>% 
  rownames_to_column(var = "Isolate") %>% 
  left_join(corrected_loci, by = "Isolate")
stopifnot(identical(datdf$Isolate, indNames(dat)))
datdf <- datdf[c("Isolate", names(repeat_lengths))]
dat   <- datdf %>% 
  select(-Isolate) %>% 
  df2genind(ind.names = indNames(dat), strata = strata(dat), ploidy = 1) %>% 
  as.genclone()

strata(dat) %>%
  bind_cols(datdf) %>%
  dplyr::as_data_frame() %>%
  readr::write_csv("data/clean_data.csv", col_names = TRUE)
```

The original data includes both Severity and Isolate. Since these are not
necessary for delimiting the strata, we will place them in the "other" slot
after converting Severity to numeric. Placing this information in the "other"
slot ensures that these data will travel with the object.

> Note 2017-06-29: I realized that the severity data was not present in the
> clean data, so I added that in the strata above. To avoid downstream effects,
> I'm additionally removing it from the data set here:


```r
stopifnot(identical(indNames(dat), saj$Isolate))
other(dat)$meta <- select(saj, Severity, Isolate)
strata(dat) <- select(strata(dat), -Severity)
other(dat)$REPLEN <- fix_replen(dat, repeat_lengths)
```

```
## Warning in fix_replen(dat, repeat_lengths): The repeat lengths for 5-3(F), 36-4(F), 50-4(F), 92-4(F), 106-4(H) are not consistent.
## 
##  This might be due to inconsistent allele calls or repeat lengths that are too large.
##  Check the alleles to make sure there are no duplicated or similar alleles that might end up being the same after division.
##  
## Repeat lengths with some modification are being returned: 6-2(F), 110-4(H)
```


```r
setPop(dat) <- ~Region
dat
```

```
## 
## This is a genclone object
## -------------------------
## Genotype information:
## 
##    215 original multilocus genotypes 
##    366 haploid individuals
##     16 codominant loci
## 
## Population information:
## 
##      5 strata - MCG, Region, Source, Year, Host
##     14 populations defined - NE, NY, MN, ..., France, Mexico, ND
```

```r
locNames(dat)
```

```
##  [1] "5-2(F)"   "5-3(F)"   "6-2(F)"   "7-2(F)"   "8-3(H)"   "9-2(F)"  
##  [7] "12-2(H)"  "17-3(H)"  "20-3(F)"  "36-4(F)"  "50-4(F)"  "55-4(F)" 
## [13] "92-4(F)"  "106-4(H)" "110-4(H)" "114-4(H)"
```

```r
other(dat)$REPLEN
```

```
##   5-2(F)   5-3(F)   6-2(F)   7-2(F)   8-3(H)   9-2(F)  12-2(H)  17-3(H) 
##  2.00000  4.00000  5.99999  2.00000  2.00000  2.00000  2.00000  3.00000 
##  20-3(F)  36-4(F)  50-4(F)  55-4(F)  92-4(F) 106-4(H) 110-4(H) 114-4(H) 
##  2.00000  4.00000  4.00000  4.00000  2.00000  4.00000  3.99999  4.00000
```

```r
head(other(dat)$meta)
```

```
## # A tibble: 6 x 2
##   Severity Isolate
##      <dbl>   <chr>
## 1      3.9     152
## 2      5.4     274
## 3      6.3     443
## 4      4.4     444
## 5      4.7     445
## 6      6.1     446
```

```r
keeploci <- !locNames(dat) %in% colnames(corrected_loci)
dat11 <- dat[loc = keeploci, mlg.reset = TRUE] # reducing to 11 loci and recalculating mlgs
dat11
```

```
## 
## This is a genclone object
## -------------------------
## Genotype information:
## 
##    165 original multilocus genotypes 
##    366 haploid individuals
##     11 codominant loci
## 
## Population information:
## 
##      5 strata - MCG, Region, Source, Year, Host
##     14 populations defined - NE, NY, MN, ..., France, Mexico, ND
```




```r
save(dat, dat11, datdf, keeploci, corrected_loci, file = "data/sclerotinia_16_loci.rda")
```


## Session Information


```r
options(width = 100)
devtools::session_info()
```

```
## Session info --------------------------------------------------------------------------------------
```

```
##  setting  value                       
##  version  R version 3.4.2 (2017-09-28)
##  system   x86_64, linux-gnu           
##  ui       X11                         
##  language (EN)                        
##  collate  en_US.UTF-8                 
##  tz       UTC                         
##  date     2018-04-12
```

```
## Packages ------------------------------------------------------------------------------------------
```

```
##  package     * version date       source         
##  ade4        * 1.7-8   2017-08-09 cran (@1.7-8)  
##  adegenet    * 2.1.0   2017-10-12 cran (@2.1.0)  
##  ape           5.0     2017-10-30 cran (@5.0)    
##  assertr     * 2.0.2.2 2017-06-06 cran (@2.0.2.2)
##  assertthat    0.2.0   2017-04-11 CRAN (R 3.4.2) 
##  base        * 3.4.2   2018-03-01 local          
##  bindr         0.1     2016-11-13 CRAN (R 3.4.2) 
##  bindrcpp    * 0.2     2017-06-17 CRAN (R 3.4.2) 
##  boot          1.3-20  2017-07-30 cran (@1.3-20) 
##  broom         0.4.3   2017-11-20 CRAN (R 3.4.2) 
##  cellranger    1.1.0   2016-07-27 CRAN (R 3.4.2) 
##  cli           1.0.0   2017-11-05 CRAN (R 3.4.2) 
##  cluster       2.0.6   2017-03-16 CRAN (R 3.4.2) 
##  coda          0.19-1  2016-12-08 cran (@0.19-1) 
##  colorspace    1.3-2   2016-12-14 CRAN (R 3.4.2) 
##  compiler      3.4.2   2018-03-01 local          
##  crayon        1.3.4   2017-09-16 CRAN (R 3.4.2) 
##  datasets    * 3.4.2   2018-03-01 local          
##  deldir        0.1-14  2017-04-22 cran (@0.1-14) 
##  devtools      1.13.4  2017-11-09 CRAN (R 3.4.2) 
##  digest        0.6.12  2017-01-27 CRAN (R 3.4.2) 
##  dplyr       * 0.7.4   2017-09-28 CRAN (R 3.4.2) 
##  evaluate      0.10.1  2017-06-24 CRAN (R 3.4.2) 
##  expm          0.999-2 2017-03-29 cran (@0.999-2)
##  ezknitr       0.6     2016-09-16 cran (@0.6)    
##  fastmatch     1.1-0   2017-01-28 cran (@1.1-0)  
##  forcats     * 0.2.0   2017-01-23 CRAN (R 3.4.2) 
##  foreign       0.8-69  2017-06-21 CRAN (R 3.4.2) 
##  gdata         2.18.0  2017-06-06 cran (@2.18.0) 
##  ggplot2     * 2.2.1   2016-12-30 CRAN (R 3.4.2) 
##  glue          1.2.0   2017-10-29 CRAN (R 3.4.2) 
##  gmodels       2.16.2  2015-07-22 cran (@2.16.2) 
##  graphics    * 3.4.2   2018-03-01 local          
##  grDevices   * 3.4.2   2018-03-01 local          
##  grid          3.4.2   2018-03-01 local          
##  gtable        0.2.0   2016-02-26 CRAN (R 3.4.2) 
##  gtools        3.5.0   2015-05-29 cran (@3.5.0)  
##  haven         1.1.0   2017-07-09 CRAN (R 3.4.2) 
##  hms           0.4.0   2017-11-23 CRAN (R 3.4.2) 
##  htmltools     0.3.6   2017-04-28 CRAN (R 3.4.2) 
##  httpuv        1.3.5   2017-07-04 CRAN (R 3.4.2) 
##  httr          1.3.1   2017-08-20 CRAN (R 3.4.2) 
##  igraph        1.1.2   2017-07-21 CRAN (R 3.4.2) 
##  jsonlite      1.5     2017-06-01 CRAN (R 3.4.2) 
##  knitr       * 1.17    2017-08-10 CRAN (R 3.4.2) 
##  lattice       0.20-35 2017-03-25 CRAN (R 3.4.2) 
##  lazyeval      0.2.1   2017-10-29 CRAN (R 3.4.2) 
##  LearnBayes    2.15    2014-05-29 cran (@2.15)   
##  lubridate     1.7.1   2017-11-03 CRAN (R 3.4.2) 
##  magrittr      1.5     2014-11-22 CRAN (R 3.4.2) 
##  MASS          7.3-47  2017-04-21 CRAN (R 3.4.2) 
##  Matrix        1.2-12  2017-11-16 CRAN (R 3.4.2) 
##  memoise       1.1.0   2017-04-21 CRAN (R 3.4.2) 
##  methods     * 3.4.2   2018-03-01 local          
##  mgcv          1.8-22  2017-09-19 CRAN (R 3.4.2) 
##  mime          0.5     2016-07-07 CRAN (R 3.4.2) 
##  mnormt        1.5-5   2016-10-15 CRAN (R 3.4.2) 
##  modelr        0.1.1   2017-07-24 CRAN (R 3.4.2) 
##  munsell       0.4.3   2016-02-13 CRAN (R 3.4.2) 
##  nlme          3.1-131 2017-02-06 CRAN (R 3.4.2) 
##  parallel      3.4.2   2018-03-01 local          
##  pegas         0.10    2017-05-03 cran (@0.10)   
##  permute       0.9-4   2016-09-09 cran (@0.9-4)  
##  phangorn      2.3.1   2017-11-01 cran (@2.3.1)  
##  pkgconfig     2.0.1   2017-03-21 CRAN (R 3.4.2) 
##  plyr          1.8.4   2016-06-08 CRAN (R 3.4.2) 
##  poppr       * 2.5.0   2017-09-11 cran (@2.5.0)  
##  psych         1.7.8   2017-09-09 CRAN (R 3.4.2) 
##  purrr       * 0.2.4   2017-10-18 CRAN (R 3.4.2) 
##  quadprog      1.5-5   2013-04-17 cran (@1.5-5)  
##  R.methodsS3   1.7.1   2016-02-16 cran (@1.7.1)  
##  R.oo          1.21.0  2016-11-01 cran (@1.21.0) 
##  R.utils       2.6.0   2017-11-05 cran (@2.6.0)  
##  R6            2.2.2   2017-06-17 CRAN (R 3.4.2) 
##  Rcpp          0.12.14 2017-11-23 CRAN (R 3.4.2) 
##  readr       * 1.1.1   2017-05-16 CRAN (R 3.4.2) 
##  readxl      * 1.0.0   2017-04-18 CRAN (R 3.4.2) 
##  reshape2      1.4.2   2016-10-22 CRAN (R 3.4.2) 
##  rlang         0.1.4   2017-11-05 CRAN (R 3.4.2) 
##  rstudioapi    0.7     2017-09-07 CRAN (R 3.4.2) 
##  rvest         0.3.2   2016-06-17 CRAN (R 3.4.2) 
##  scales        0.5.0   2017-08-24 CRAN (R 3.4.2) 
##  seqinr        3.4-5   2017-08-01 cran (@3.4-5)  
##  shiny         1.0.5   2017-08-23 CRAN (R 3.4.2) 
##  sp            1.2-5   2017-06-29 CRAN (R 3.4.2) 
##  spData        0.2.6.7 2017-11-28 cran (@0.2.6.7)
##  spdep         0.7-4   2017-11-22 cran (@0.7-4)  
##  splines       3.4.2   2018-03-01 local          
##  stats       * 3.4.2   2018-03-01 local          
##  stringi       1.1.6   2017-11-17 CRAN (R 3.4.2) 
##  stringr     * 1.2.0   2017-02-18 CRAN (R 3.4.2) 
##  tibble      * 1.3.4   2017-08-22 CRAN (R 3.4.2) 
##  tidyr       * 0.7.2   2017-10-16 CRAN (R 3.4.2) 
##  tidyselect    0.2.3   2017-11-06 CRAN (R 3.4.2) 
##  tidyverse   * 1.2.1   2017-11-14 CRAN (R 3.4.2) 
##  tools         3.4.2   2018-03-01 local          
##  utils       * 3.4.2   2018-03-01 local          
##  vegan         2.4-4   2017-08-24 cran (@2.4-4)  
##  withr         2.1.0   2017-11-01 CRAN (R 3.4.2) 
##  xml2          1.1.1   2017-01-24 CRAN (R 3.4.2) 
##  xtable        1.8-2   2016-02-05 CRAN (R 3.4.2)
```

