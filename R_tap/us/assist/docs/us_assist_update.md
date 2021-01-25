Federal Financial Assistance Update
================
Kiernan Nicholls
2020-07-17 12:18:32

  - [Project](#project)
  - [Objectives](#objectives)
  - [Software](#software)
  - [Data](#data)
  - [Download](#download)
  - [Extract](#extract)
  - [Layout](#layout)
  - [Read](#read)
  - [Check](#check)
  - [Delta](#delta)

<!-- Place comments regarding knitting here -->

## Project

The Accountability Project is an effort to cut across data silos and
give journalists, policy professionals, activists, and the public at
large a simple way to search across huge volumes of public data about
people and organizations.

Our goal is to standardizing public data on a few key fields by thinking
of each dataset row as a transaction. For each transaction there should
be (at least) 3 variables:

1.  All **parties** to a transaction.
2.  The **date** of the transaction.
3.  The **amount** of money involved.

## Objectives

This document describes the process used to complete the following
objectives:

1.  How many records are in the database?
2.  Check for entirely duplicated records.
3.  Check ranges of continuous variables.
4.  Is there anything blank or missing?
5.  Check for consistency issues.
6.  Create a five-digit ZIP Code called `zip`.
7.  Create a `year` field from the transaction date.
8.  Make sure there is data on both parties to a transaction.

## Software

This data is processed using the free, open-source statistical computing
language R, which can be [installed from
CRAN](https://cran.r-project.org/) for various opperating systems. For
example, R can be installed from the apt package repository on Ubuntu.

``` bash
sudo apt update
sudo apt -y upgrade
sudo apt -y install r-base
```

The following additional R packages are needed to collect, manipulate,
visualize, analyze, and communicate these results. The `pacman` package
will facilitate their installation and attachment.

The IRW’s `campfin` package will also have to be installed from GitHub.
This package contains functions custom made to help facilitate the
processing of campaign finance data.

``` r
if (!require("pacman")) install.packages("pacman")
pacman::p_load_gh("irworkshop/campfin")
pacman::p_load(
  tidyverse, # data manipulation
  lubridate, # datetime strings
  magrittr, # pipe operators
  gluedown, # print markdown
  janitor, # dataframe clean
  refinr, # cluster and merge
  scales, # format strings
  readxl, # read excel
  knitr, # knit documents
  vroom, # read files fast
  furrr, # parallel map
  glue, # combine strings
  here, # relative storage
  pryr, # memory usage
  fs # search storage 
)
```

This document should be run as part of the `us_spending` project, which
lives as a sub-directory of the more general, language-agnostic
[`irworkshop/tap`](https://github.com/irworkshop/accountability_datacleaning)
GitHub repository.

The `us_spending` project uses the [RStudio
projects](https://support.rstudio.com/hc/en-us/articles/200526207-Using-Projects)
feature and should be run as such. The project also uses the dynamic
`here::here()` tool for file paths relative to *your* machine.

``` r
# where does this document knit?
here::here()
#> [1] "/home/kiernan/Code/tap/R_campfin"
```

## Data

Federal spending data is obtained from
[USASpending.gov](https://www.usaspending.gov/#/), a site run by the
Department of the Treasury.

> \[Many\] sources of information support USAspending.gov, linking data
> from a variety of government systems to improve transparency on
> federal spending for the public. Data is uploaded directly from more
> than a hundred federal agencies’ financial systems. Data is also
> pulled or derived from other government systems… In the end, more than
> 400 points of data are collected…

> Federal agencies submit contract, grant, loan, direct payment, and
> other award data at least twice a month to be published on
> USAspending.gov. Federal agencies upload data from their financial
> systems and link it to the award data quarterly. This quarterly data
> must be certified by the agency’s Senior Accountable Official before
> it is displayed on USAspending.gov.

Flat text files containing all spending data can be found on the [Award
Data
Archive](https://www.usaspending.gov/#/download_center/award_data_archive).

> Welcome to the Award Data Archive, which features major agencies’
> award transaction data for full fiscal years. They’re a great way to
> get a view into broad spending trends and, best of all, the files are
> already prepared — you can access them instantaneously.

In this document, we are only going to update the most recent file to
ensure the data in on the TAP website is up to date. Instead of
downloading multiple years ZIP archives, we will only download the
current year and the delta file.

Archives can be obtained for individual agencies or for *all* agencies.

## Download

We first need to construct both the URLs and local paths to the archive
files.

``` r
zip_dir <- dir_create(here("us", "assist", "data", "zip"))
base_url <- "https://files.usaspending.gov/award_data_archive/"
fin_files <- "FY2020_All_Assistance_Full_20200713.zip"
fin_urls <- str_c(base_url, fin_files)
fin_zips <- path(zip_dir, fin_files)
```

    #> [1] "https://files.usaspending.gov/award_data_archive/FY2020_All_Assistance_Full_20200713.zip"
    #> [1] "~/us/assist/data/zip/FY2020_All_Assistance_Full_20200713.zip"

We also need to add the records for spending and corrections made since
this file was last updated. This is information is crucial, as it
contains the most recent data. This information can be found in the
“delta” file released alongside the “full” spending files.

> New files are uploaded by the 15th of each month. Check the Data As Of
> column to see the last time files were generated. Full files feature
> data for the fiscal year up until the date the file was prepared, and
> delta files feature only new, modified, and deleted data since the
> date the last month’s files were generated. The
> `correction_delete_ind` column in the delta files indicates whether a
> record has been modified (C), deleted (D), or added (blank). To
> download data prior to FY 2008, visit our Custom Award Data page.

``` r
delta_file <- "FY(All)_All_Assistance_Delta_20200713.zip"
delta_url <- str_c(base_url, delta_file)
delta_zip <- path(zip_dir, delta_file)
```

These files are large, so we might want to check their size before
downloading.

``` r
(fin_size <- tibble(
  url = basename(fin_urls),
  size = as_fs_bytes(map_dbl(fin_urls, url_file_size))
))
#> # A tibble: 1 x 2
#>   url                                            size
#>   <chr>                                   <fs::bytes>
#> 1 FY2020_All_Assistance_Full_20200713.zip        567M
```

``` r
if (require(speedtest)) {
  # remotes::install_github("hrbrmstr/speedtest")
  config <- speedtest::spd_config()
  servers <- speedtest::spd_servers(config = config)
  closest_servers <- speedtest::spd_closest_servers(servers, config = config)
  speed <- speedtest::spd_download_test(closest_servers[1, ], config = config)
  # use median results
  speed[, 11:15]
  # minutes to download
  ((sum(fin_size$size)/1e+6) / (speed$median/8))/60
}
```

If the archive files have not been downloaded, we can do so now.

``` r
if (!all(file_exists(c(fin_zips, delta_zip)))) {
  download.file(fin_urls, fin_zips)
  download.file(delta_url, delta_zip)
}
```

## Extract

We can extract the text files from the annual archives into a new
directory.

``` r
raw_dir <- dir_create(here("us", "assist", "data", "raw"))
if (length(dir_ls(raw_dir)) == 0) {
  future_map(fin_zips, unzip, exdir = raw_dir)
  future_map(delta_zip, unzip, exdir = raw_dir)
}
```

``` r
fin_paths <- dir_ls(raw_dir, regexp = "FY\\d+.*csv")
delta_paths <- dir_ls(raw_dir, regexp = "FY\\(All\\).*csv")
```

## Layout

The USA Spending website also provides a comprehensive data dictionary
which covers the many variables in this file.

``` r
dict_file <- path(here("us", "assist", "data"), "dict.xlsx")
if (!file_exists(dict_file)) {
  download.file(
    url = "https://files.usaspending.gov/docs/Data_Dictionary_Crosswalk.xlsx",
    destfile = dict_file
  )
}
dict <- read_excel(
  path = dict_file, 
  range = "A2:L414",
  na = "N/A",
  .name_repair = make_clean_names
)

usa_names <- names(vroom(fin_paths[which.min(file_size(fin_paths))], n_max = 0))
# get cols from hhs data
mean(usa_names %in% dict$award_element)
#> [1] NaN
dict %>% 
  filter(award_element %in% usa_names) %>% 
  select(award_element, definition) %>% 
  mutate_at(vars(definition), str_replace_all, "\"", "\'") %>% 
  arrange(match(award_element, usa_names)) %>% 
  head(10) %>% 
  mutate_at(vars(definition), str_trunc, 75) %>% 
  kable()
```

| award\_element | definition |
| :------------- | :--------- |

## Read

Due to the sheer size and number of files in question, we can’t read
them all at once into a single data file for exploration and wrangling.

``` r
length(fin_paths)
#> [1] 0
# total file sizes
sum(file_size(fin_paths))
#> 0
# avail local memory
as_fs_bytes(str_extract(system("free", intern = TRUE)[2], "\\d+"))
#> 31.4M
```

What we will instead do is read each file individually and perform the
type of exploratory analysis we need to ensure the data is well
structured and normal. This will be done with a lengthy `for` loop and
appending the checks to a new text file on disk.

We can append the checks from the new updated files onto the checks from
the entire database (going back to 2008) and separate them with a
comment line.

``` r
spend_path <- here("us", "assist", "spend_check.csv")
write_lines("###### updates", spend_path, append = TRUE)
```

We are not going to use the delta file to correct, delete, and update
the original transactions. We are instead going to upload the separately
so that the changed versions appear alongside the original in all search
results. We will tag all records with the file they originate from.

``` r
# track progress in text file
prog_path <- file_create(here("us", "assist", "read_prog.txt"))
for (f in c(fin_paths, delta_paths)) {
  prog_files <- read_lines(prog_path)
  n <- str_remove(basename(f), "_All_Assistance_(Full|Delta)_\\d+")
  if (f %in% prog_files) {
    message(paste(n, "already done"))
    next()
  } else {
    message(paste(n, "starting"))
  }
  # read contracts ------------------------------------------------------------
  usc <- vroom(
    file = f,
    delim = ",",
    guess_max = 0,
    escape_backslash = FALSE,
    escape_double = FALSE,
    progress = FALSE,
    id = "file",
    num_threads = 1,
    col_types = cols(
      .default = col_character(),
      action_date_fiscal_year = col_integer(),
      action_date = col_date(),
      federal_action_obligation = col_double()
    )
  )
  usc <- select(
    .data = usc,
    key = assistance_award_unique_key,
    piid = award_id_fain,
    fiscal = action_date_fiscal_year,
    date = action_date,
    amount = federal_action_obligation,
    agency = awarding_agency_name,
    sub_id = awarding_sub_agency_code,
    sub_agency = awarding_sub_agency_name,
    office = awarding_office_name,
    rec_id = recipient_duns,
    address1 = recipient_address_line_1,
    address2 = recipient_address_line_2,
    city = recipient_city_name,
    state = recipient_state_code,
    zip = recipient_zip_code,
    place = primary_place_of_performance_zip_4,
    type = assistance_type_code,
    desc = award_description,
    file,
    everything()
  )
  # tweak cols ---------------------------------------------------------------
  usc <- mutate( # create single recip col
    .data = usc,
    .after = "rec_id", 
    file = basename(file),
    rec_name = coalesce(
      recipient_name,
      recipient_parent_name
    )
  )
  usc <- flag_na(usc, date, amount, sub_agency, rec_name) # flag missing vals
  usc <- mutate_at(usc, vars("zip", "place"), str_sub, end = 5) # trim zip
  usc <- mutate(usc, year = year(date), .after = "fiscal") # add calendar year
  flush_memory()
  # if delta remove rows
  if ("correction_delete_ind" %in% names(usc)) {
    usc <- rename(usc, change = correction_delete_ind)
    usc <- relocate(usc, change, .after = "file")
    usc <- filter(usc, change != "D" | is.na(change))
  }
  # save checks --------------------------------------------------------------
  if (n_distinct(usc$fiscal) > 1) {
    fy <- NA_character_
  } else {
    fy <- unique(usc$fiscal)
  }
  check <- tibble(
    file = n,
    nrow = nrow(usc),
    ncol = ncol(usc),
    types = n_distinct(usc$type),
    fiscal = fy,
    start = min(usc$date, na.rm = TRUE),
    end = max(usc$date, na.rm = TRUE),
    miss = sum(usc$na_flag, na.rm = TRUE),
    zero = sum(usc$amount <= 0, na.rm = TRUE),
    city = round(prop_in(usc$city, c(valid_city, extra_city)), 4),
    state = round(prop_in(usc$state, valid_state), 4),
    zip = round(prop_in(usc$zip, valid_zip), 4)
  )
  message(paste(n, "checking done"))
  vroom_write(x = usc, path = f, delim = ",", na = "") # save manipulated file
  write_csv(check[1, ], spend_path, append = TRUE) # save the checks as line 
  write_lines(f, prog_path, append = TRUE) # save the file as line
  # reset for next
  rm(usc, check) 
  flush_memory(n = 100)
  p <- paste(match(f, fin_paths), length(fin_paths), sep = "/")
  message(paste(n, "writing done:", p, file_size(f)))
  beepr::beep("fanfare")
  Sys.sleep(30)
}
```

## Check

In the end, 8 files were read and checked.

``` r
all_paths <- dir_ls(raw_dir)
length(all_paths)
#> [1] 8
sum(file_size(all_paths))
#> 5.97G
```

Now we can read the `spend_check.csv` text file to see the statistics
saved from each file.

We can `summarise()` across all files to find the typical statistic
across all raw data.

``` r
all_checks %>% 
  summarise(
    nrow = sum(nrow),
    ncol = mean(ncol),
    type = mean(types),
    start = min(start),
    end = max(end),
    missing = sum(missing)/sum(nrow),
    zero = sum(zero)/sum(nrow),
    city = mean(city),
    state = mean(state),
    zip = mean(zip)
  )
#> # A tibble: 1 x 10
#>       nrow  ncol  type start      end        missing  zero  city state   zip
#>      <dbl> <dbl> <dbl> <date>     <date>       <dbl> <dbl> <dbl> <dbl> <dbl>
#> 1 61879267  29.3  9.94 2006-01-25 2020-09-30 0.00395 0.244 0.991  1.00 0.999
```

``` r
per_day <- all_checks %>% 
  group_by(file) %>% 
  arrange(desc(nrow)) %>% 
  slice(1) %>% 
  group_by(fiscal) %>% 
  summarise(nrow = sum(nrow)) %>% 
  mutate(per = nrow/365)
per_day$per[13] <- per_day$nrow[13]/yday(today())
per_day %>% 
  ggplot(aes(fiscal, per)) + 
  geom_col(fill = dark2["purple"]) +
  scale_y_continuous(labels = comma) +
  labs(
    title = "US Spending Transactions per day by year",
    subtitle = glue("As of {today()}"),
    x = "Fiscal Year",
    y = "Unique Transactions"
  )
```

![](../plots/year_bar-1.png)<!-- -->

And here we have the total checks for every file.

| file            | nrow      | start      | end        | missing | zero  | city  | state  | zip    |
| :-------------- | :-------- | :--------- | :--------- | ------: | :---- | :---- | :----- | :----- |
| `FY2008_1.csv`  | 1,000,000 | 2008-03-31 | 2008-09-30 |       0 | 57.7% | 98.7% | 99.9%  | 99.8%  |
| `FY2008_2.csv`  | 608,006   | 2007-10-01 | 2008-03-31 |       0 | 62.0% | 98.8% | 100.0% | 99.6%  |
| `FY2009_1.csv`  | 1,000,000 | 2009-06-18 | 2009-09-30 |       0 | 51.3% | 98.3% | 99.6%  | 99.7%  |
| `FY2009_2.csv`  | 1,000,000 | 2009-02-05 | 2009-06-18 |       0 | 63.8% | 98.9% | 99.9%  | 99.8%  |
| `FY2009_3.csv`  | 577,111   | 2008-10-01 | 2009-02-05 |       0 | 61.2% | 98.9% | 99.9%  | 99.8%  |
| `FY2010_1.csv`  | 1,000,000 | 2010-06-24 | 2010-09-30 |       0 | 53.9% | 98.6% | 99.6%  | 99.8%  |
| `FY2010_2.csv`  | 1,000,000 | 2010-04-02 | 2010-06-24 |       0 | 50.5% | 99.1% | 99.8%  | 99.8%  |
| `FY2010_3.csv`  | 1,000,000 | 2010-01-18 | 2010-04-02 |       0 | 46.4% | 99.2% | 99.9%  | 99.8%  |
| `FY2010_4.csv`  | 1,000,000 | 2009-10-23 | 2010-01-18 |       0 | 38.0% | 99.1% | 99.9%  | 99.8%  |
| `FY2010_5.csv`  | 579,794   | 2009-10-01 | 2009-10-23 |       0 | 14.8% | 99.2% | 100.0% | 100.0% |
| `FY2011_1.csv`  | 1,000,000 | 2011-06-30 | 2011-09-30 |       0 | 39.7% | 98.7% | 99.5%  | 99.8%  |
| `FY2011_2.csv`  | 1,000,000 | 2011-04-18 | 2011-06-30 |       0 | 46.8% | 99.2% | 99.7%  | 99.9%  |
| `FY2011_3.csv`  | 1,000,000 | 2011-01-01 | 2011-04-18 |       0 | 42.6% | 99.1% | 99.8%  | 99.9%  |
| `FY2011_4.csv`  | 1,000,000 | 2010-10-09 | 2011-01-01 |       0 | 37.0% | 99.2% | 100.0% | 99.9%  |
| `FY2011_5.csv`  | 446,187   | 2010-10-01 | 2010-10-09 |       0 | 8.9%  | 99.1% | 100.0% | 100.0% |
| `FY2012_1.csv`  | 1,000,000 | 2012-05-29 | 2012-09-30 |       0 | 23.7% | 98.3% | 100.0% | 99.7%  |
| `FY2012_2.csv`  | 1,000,000 | 2011-12-31 | 2012-05-29 |       0 | 22.0% | 98.7% | 99.6%  | 99.7%  |
| `FY2012_3.csv`  | 1,000,000 | 2011-10-07 | 2011-12-31 |       0 | 18.0% | 99.1% | 99.7%  | 99.8%  |
| `FY2012_4.csv`  | 274,924   | 2011-10-01 | 2011-10-07 |       0 | 5.5%  | 99.0% | 100.0% | 100.0% |
| `FY2013_1.csv`  | 1,000,000 | 2013-06-07 | 2013-09-30 |       0 | 22.8% | 97.8% | 100.0% | 99.9%  |
| `FY2013_2.csv`  | 1,000,000 | 2013-01-29 | 2013-06-07 |       0 | 23.0% | 98.6% | 100.0% | 99.6%  |
| `FY2013_3.csv`  | 1,000,000 | 2012-10-05 | 2013-01-29 |       0 | 19.4% | 99.0% | 100.0% | 98.5%  |
| `FY2013_4.csv`  | 436,000   | 2012-10-01 | 2012-10-05 |       0 | 0.7%  | 99.0% | 100.0% | 100.0% |
| `FY2014_1.csv`  | 1,000,000 | 2014-06-27 | 2014-09-30 |       0 | 20.3% | 97.8% | 100.0% | 99.9%  |
| `FY2014_2.csv`  | 1,000,000 | 2014-02-28 | 2014-06-27 |       0 | 23.6% | 98.1% | 100.0% | 99.9%  |
| `FY2014_3.csv`  | 1,000,000 | 2013-10-31 | 2014-02-28 |       0 | 22.6% | 98.7% | 100.0% | 100.0% |
| `FY2014_4.csv`  | 818,183   | 2013-10-01 | 2013-10-31 |       0 | 6.0%  | 99.0% | 100.0% | 100.0% |
| `FY2015_1.csv`  | 1,000,000 | 2015-06-17 | 2015-09-30 |       0 | 20.8% | 97.4% | 100.0% | 99.9%  |
| `FY2015_2.csv`  | 1,000,000 | 2015-02-10 | 2015-06-17 |       0 | 22.9% | 98.2% | 100.0% | 99.9%  |
| `FY2015_3.csv`  | 1,000,000 | 2014-10-28 | 2015-02-10 |       0 | 21.2% | 98.6% | 100.0% | 100.0% |
| `FY2015_4.csv`  | 328,425   | 2014-10-01 | 2014-10-28 |       0 | 13.5% | 98.9% | 100.0% | 100.0% |
| `FY2016_1.csv`  | 1,000,000 | 2016-06-28 | 2016-09-30 |       0 | 19.3% | 97.8% | 100.0% | 99.9%  |
| `FY2016_2.csv`  | 1,000,000 | 2016-02-25 | 2016-06-28 |       0 | 22.8% | 98.1% | 100.0% | 99.9%  |
| `FY2016_3.csv`  | 1,000,000 | 2015-11-03 | 2016-02-25 |       0 | 20.2% | 98.2% | 100.0% | 100.0% |
| `FY2016_4.csv`  | 632,643   | 2015-10-01 | 2015-11-03 |       0 | 8.1%  | 99.0% | 100.0% | 100.0% |
| `FY2017_1.csv`  | 1,000,000 | 2017-06-30 | 2017-09-30 |       0 | 14.6% | 99.4% | 100.0% | 100.0% |
| `FY2017_2.csv`  | 1,000,000 | 2017-03-23 | 2017-06-30 |       0 | 17.4% | 99.5% | 100.0% | 100.0% |
| `FY2017_3.csv`  | 1,000,000 | 2016-12-05 | 2017-03-23 |       0 | 19.2% | 99.3% | 100.0% | 100.0% |
| `FY2017_4.csv`  | 887,842   | 2016-10-01 | 2016-12-05 |       0 | 15.5% | 99.0% | 100.0% | 100.0% |
| `FY2018_1.csv`  | 1,000,000 | 2017-10-01 | 2018-09-30 |       0 | 8.6%  | 99.6% | 100.0% | 100.0% |
| `FY2018_2.csv`  | 1,000,000 | 2017-10-01 | 2018-09-30 |       0 | 8.8%  | 99.6% | 100.0% | 100.0% |
| `FY2018_3.csv`  | 1,000,000 | 2017-10-01 | 2018-09-30 |       0 | 8.6%  | 99.6% | 100.0% | 100.0% |
| `FY2018_4.csv`  | 1,000,000 | 2017-10-01 | 2018-09-30 |       0 | 8.8%  | 99.5% | 100.0% | 100.0% |
| `FY2018_5.csv`  | 1,000,000 | 2017-10-01 | 2018-09-30 |       0 | 8.9%  | 99.5% | 100.0% | 100.0% |
| `FY2018_6.csv`  | 1,000,000 | 2017-10-01 | 2018-09-30 |       0 | 8.5%  | 99.5% | 100.0% | 100.0% |
| `FY2018_7.csv`  | 1,000,000 | 2017-10-01 | 2018-09-30 |       0 | 8.8%  | 99.6% | 100.0% | 100.0% |
| `FY2018_8.csv`  | 263,073   | 2017-10-01 | 2018-09-30 |       0 | 9.2%  | 99.5% | 100.0% | 100.0% |
| `FY2019_1.csv`  | 1,000,000 | 2018-10-01 | 2019-09-30 |       0 | 17.9% | 99.6% | 100.0% | 100.0% |
| `FY2019_2.csv`  | 1,000,000 | 2018-10-01 | 2019-09-30 |       0 | 18.2% | 99.6% | 100.0% | 100.0% |
| `FY2019_3.csv`  | 1,000,000 | 2018-10-01 | 2019-09-30 |       0 | 18.4% | 99.6% | 100.0% | 100.0% |
| `FY2019_4.csv`  | 1,000,000 | 2018-10-01 | 2019-09-30 |       0 | 18.2% | 99.6% | 100.0% | 100.0% |
| `FY2019_5.csv`  | 1,000,000 | 2018-10-01 | 2019-09-30 |       0 | 18.2% | 99.6% | 100.0% | 100.0% |
| `FY2019_6.csv`  | 1,000,000 | 2018-10-01 | 2019-09-30 |       0 | 18.2% | 99.6% | 100.0% | 100.0% |
| `FY2019_7.csv`  | 1,000,000 | 2018-10-01 | 2019-09-30 |       0 | 18.2% | 99.6% | 100.0% | 100.0% |
| `FY2019_8.csv`  | 843,036   | 2018-10-01 | 2019-09-30 |       0 | 18.2% | 99.6% | 100.0% | 100.0% |
| `FY2020_1.csv`  | 1,000,000 | 2019-12-02 | 2020-09-30 |       0 | 41.6% | 99.7% | 100.0% | 100.0% |
| `FY2020_2.csv`  | 1,000,000 | 2019-10-17 | 2019-12-02 |       0 | 22.7% | 99.6% | 100.0% | 100.0% |
| `FY2020_3.csv`  | 1,000,000 | 2019-10-08 | 2019-10-17 |       0 | 5.1%  | 99.5% | 100.0% | 100.0% |
| `FY2020_4.csv`  | 1,000,000 | 2019-10-04 | 2019-10-08 |       0 | 1.4%  | 99.4% | 100.0% | 100.0% |
| `FY2020_5.csv`  | 74,967    | 2019-10-01 | 2019-10-04 |       0 | 27.3% | 99.5% | 100.0% | 100.0% |
| `FY(All)_1.csv` | 985,905   | 2007-07-13 | 2020-09-30 |       0 | 41.3% | 99.7% | 100.0% | 100.0% |
| `FY(All)_1.csv` | 985,905   | 2007-07-13 | 2020-09-30 |       0 | 41.3% | 99.7% | 100.0% | 100.0% |
| `FY2020_1.csv`  | 1,000,000 | 2020-03-18 | 2020-09-30 |   43897 | 25.6% | 99.7% | 100.0% | 100.0% |
| `FY2020_2.csv`  | 1,000,000 | 2020-01-07 | 2020-03-18 |   37265 | 43.5% | 99.8% | 100.0% | 100.0% |
| `FY2020_3.csv`  | 1,000,000 | 2019-11-11 | 2020-01-07 |   54806 | 33.0% | 99.7% | 100.0% | 100.0% |
| `FY2020_4.csv`  | 1,000,000 | 2019-10-09 | 2019-11-11 |   62183 | 16.6% | 99.5% | 100.0% | 100.0% |
| `FY2020_5.csv`  | 1,000,000 | 2019-10-08 | 2019-10-09 |    1397 | 0.6%  | 99.4% | 100.0% | 100.0% |
| `FY2020_6.csv`  | 701,666   | 2019-10-01 | 2019-10-08 |    7675 | 4.7%  | 99.4% | 100.0% | 100.0% |
| `FY(All)_1.csv` | 435,600   | 2006-01-25 | 2020-07-10 |   37432 | 28.8% | 99.8% | 100.0% | 100.0% |

## Delta

These 2020 files contain *all* records of contracts made in the 2020
fiscal year as of June 12. This means records that were previously only
listed in th Delta files are now incorporated in the regular dataset. To
ensure the TAP database is both comprehensive and absent of needless
duplicates, we will remove the previous delta file and replace it only
with those records not now found in the regular data.

We will have to manually download the old delta files and read them into
a single data frame. Then, we can filter out duplicate records and
create a new file of old delta records, mostly corrections to pre-2020
records.

``` r
new_delta_path <- path(raw_dir, "us_assist_delta-old.csv")
old_delta <- vroom(dir_ls(here("us", "assist", "data", "old")))
nrow(old_delta)
#> [1] 985905
new_keys <- dir_ls(raw_dir, regex = "\\d.csv$") %>% 
  map(vroom, col_types = cols(.default = "c")) %>% 
  map_df(select, key)
new_keys <- unique(as_vector(new_keys))
length(new_keys)
#> [1] 4939470
mean(old_delta$key %in% new_keys)
#> [1] 0.9930642
old_delta <- filter(old_delta, key %out% new_keys)
nrow(old_delta)
#> [1] 6838
write_csv(old_delta, new_delta_path, na = "")
```