Federal Financial Assistance
================
Kiernan Nicholls
2020-03-31 16:59:00

  - [Project](#project)
  - [Objectives](#objectives)
  - [Software](#software)
  - [Data](#data)
  - [Download](#download)
  - [Extract](#extract)
  - [Layout](#layout)
  - [Read](#read)
  - [Check](#check)

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
#> [1] "/home/kiernan/Code/accountability_datacleaning/us_spending"
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

Data can be obtained from the archive as annual `.zip` files each
containing a number of comma-delimited text files with a maximum one
million records to reduce size.

Archives can be obtained for individual agencies or for *all* agencies.

## Download

We first need to construct both the URLs and local paths to the archive
files.

``` r
zip_dir <- dir_create(here("assist", "data", "zip"))
base_url <- "https://files.usaspending.gov/award_data_archive/"
fin_files <- glue("FY{2008:2020}_All_Assistance_Full_20200313.zip")
fin_urls <- str_c(base_url, fin_files)
fin_zips <- path(zip_dir, fin_files)
```

    #> [1] "https://files.usaspending.gov/award_data_archive/FY2008_All_Assistance_Full_20200313.zip"
    #> [2] "https://files.usaspending.gov/award_data_archive/FY2009_All_Assistance_Full_20200313.zip"
    #> [3] "https://files.usaspending.gov/award_data_archive/FY2010_All_Assistance_Full_20200313.zip"
    #> [1] "~/assist/data/zip/FY2008_All_Assistance_Full_20200313.zip"
    #> [2] "~/assist/data/zip/FY2009_All_Assistance_Full_20200313.zip"
    #> [3] "~/assist/data/zip/FY2010_All_Assistance_Full_20200313.zip"

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
delta_file <- "FY(All)_All_Assistance_Delta_20200313.zip"
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
#> # A tibble: 13 x 2
#>    url                                            size
#>    <chr>                                   <fs::bytes>
#>  1 FY2008_All_Assistance_Full_20200313.zip        174M
#>  2 FY2009_All_Assistance_Full_20200313.zip        289M
#>  3 FY2010_All_Assistance_Full_20200313.zip        505M
#>  4 FY2011_All_Assistance_Full_20200313.zip        476M
#>  5 FY2012_All_Assistance_Full_20200313.zip        340M
#>  6 FY2013_All_Assistance_Full_20200313.zip        365M
#>  7 FY2014_All_Assistance_Full_20200313.zip        385M
#>  8 FY2015_All_Assistance_Full_20200313.zip        335M
#>  9 FY2016_All_Assistance_Full_20200313.zip        373M
#> 10 FY2017_All_Assistance_Full_20200313.zip        431M
#> 11 FY2018_All_Assistance_Full_20200313.zip        626M
#> 12 FY2019_All_Assistance_Full_20200313.zip        956M
#> 13 FY2020_All_Assistance_Full_20200313.zip        371M
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
raw_dir <- dir_create(here("assist", "data", "raw"))
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
dict_file <- path(here("assist", "data"), "dict.xlsx")
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
#> [1] 0.5425532
dict %>% 
  filter(award_element %in% usa_names) %>% 
  select(award_element, definition) %>% 
  mutate_at(vars(definition), str_replace_all, "\"", "\'") %>% 
  arrange(match(award_element, usa_names)) %>% 
  head(10) %>% 
  mutate_at(vars(definition), str_trunc, 75) %>% 
  kable()
```

| award\_element                              | definition                                                                |
| :------------------------------------------ | :------------------------------------------------------------------------ |
| modification\_number                        | The identifier of an action being reported that indicates the specific s… |
| award\_id\_uri                              | Unique Record Identifier. An agency defined identifier that (when provid… |
| sai\_number                                 | A number assigned by state (as opposed to federal) review agencies to th… |
| non\_federal\_funding\_amount               | The amount of the award funded by non-Federal source(s), in dollars. Pro… |
| face\_value\_of\_loan                       | The face value of the direct loan or loan guarantee.                      |
| period\_of\_performance\_start\_date        | The date on which, for the award referred to by the action being reporte… |
| period\_of\_performance\_current\_end\_date | The current date on which, for the award referred to by the action being… |
| awarding\_agency\_code                      | A department or establishment of the Government as used in the Treasury … |
| awarding\_office\_code                      | Identifier of the level n organization that awarded, executed or is othe… |
| funding\_agency\_code                       | The 3-digit CGAC agency code of the department or establishment of the G… |

## Read

Due to the sheer size and number of files in question, we can’t read
them all at once into a single data file for exploration and wrangling.

``` r
length(fin_paths)
#> [1] 60
# total file sizes
sum(file_size(fin_paths))
#> 42.1G
# avail local memory
as_fs_bytes(str_extract(system("free", intern = TRUE)[2], "\\d+"))
#> 31.4M
```

What we will instead do is read each file individually and perform the
type of exploratory analysis we need to ensure the data is well
structured and normal. This will be done with a lengthy `for` loop and
appending the checks to a new text file on disk.

We are not going to use the delta file to correct, delete, and update
the original transactions. We are instead going to upload the separetely
so that the changed versions appear alongside the original in all search
results. We will tag all records with the file they originate from.

``` r
clear_memory <- function(n = 10) {
  for (i in 1:n) {
    gc(reset = TRUE, full = TRUE)
  }
}
```

``` r
# track progress in text file
prog_path <- file_create(here("assist", "read_prog.txt"))
spend_path <- here("assist", "spend_check.csv")
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
  clear_memory()
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
    sum = sum(usc$amount, na.rm = TRUE),
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
  clear_memory(n = 100)
  p <- paste(match(f, fin_paths), length(fin_paths), sep = "/")
  message(paste(n, "writing done:", p, file_size(f)))
  beepr::beep("fanfare")
  Sys.sleep(30)
}
```

## Check

In the end, 61 files were read and checked.

``` r
all_paths <- dir_ls(raw_dir)
length(all_paths)
#> [1] 61
sum(file_size(all_paths))
#> 43G
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
#> 1 55742001    22  9.97 2007-07-13 2020-09-30       0 0.247 0.990 0.999 0.999
```

``` r
all_checks %>% 
  group_by(fiscal) %>% 
  summarise(nrow = sum(nrow)) %>% 
  ggplot(aes(fiscal, nrow)) + 
  geom_col(fill = dark2["purple"]) +
  labs(
    title = "US Spending Transactions by Year",
    x = "Fiscal Year",
    y = "Unique Transactions"
  )
```

![](../plots/year_bar-1.png)<!-- -->

And here we have the total checks for every file.

| file            |    nrow | types | start      | end        |   zero | city  | state  | zip    |
| :-------------- | ------: | ----: | :--------- | :--------- | -----: | :---- | :----- | :----- |
| `FY2008_1.csv`  | 1000000 |    10 | 2008-03-31 | 2008-09-30 | 576642 | 98.7% | 99.9%  | 99.8%  |
| `FY2008_2.csv`  |  608006 |    10 | 2007-10-01 | 2008-03-31 | 376954 | 98.8% | 100.0% | 99.6%  |
| `FY2009_1.csv`  | 1000000 |    10 | 2009-06-18 | 2009-09-30 | 512932 | 98.3% | 99.6%  | 99.7%  |
| `FY2009_2.csv`  | 1000000 |    10 | 2009-02-05 | 2009-06-18 | 638277 | 98.9% | 99.9%  | 99.8%  |
| `FY2009_3.csv`  |  577111 |    10 | 2008-10-01 | 2009-02-05 | 353010 | 98.9% | 99.9%  | 99.8%  |
| `FY2010_1.csv`  | 1000000 |    10 | 2010-06-24 | 2010-09-30 | 539422 | 98.6% | 99.6%  | 99.8%  |
| `FY2010_2.csv`  | 1000000 |    10 | 2010-04-02 | 2010-06-24 | 505128 | 99.1% | 99.8%  | 99.8%  |
| `FY2010_3.csv`  | 1000000 |    10 | 2010-01-18 | 2010-04-02 | 463503 | 99.2% | 99.9%  | 99.8%  |
| `FY2010_4.csv`  | 1000000 |    10 | 2009-10-23 | 2010-01-18 | 380230 | 99.1% | 99.9%  | 99.8%  |
| `FY2010_5.csv`  |  579794 |    10 | 2009-10-01 | 2009-10-23 |  85910 | 99.2% | 100.0% | 100.0% |
| `FY2011_1.csv`  | 1000000 |    10 | 2011-06-30 | 2011-09-30 | 397324 | 98.7% | 99.5%  | 99.8%  |
| `FY2011_2.csv`  | 1000000 |    10 | 2011-04-18 | 2011-06-30 | 467776 | 99.2% | 99.7%  | 99.9%  |
| `FY2011_3.csv`  | 1000000 |    10 | 2011-01-01 | 2011-04-18 | 425620 | 99.1% | 99.8%  | 99.9%  |
| `FY2011_4.csv`  | 1000000 |    10 | 2010-10-09 | 2011-01-01 | 369793 | 99.2% | 100.0% | 99.9%  |
| `FY2011_5.csv`  |  446187 |    10 | 2010-10-01 | 2010-10-09 |  39775 | 99.1% | 100.0% | 100.0% |
| `FY2012_1.csv`  | 1000000 |    10 | 2012-05-29 | 2012-09-30 | 237388 | 98.3% | 100.0% | 99.7%  |
| `FY2012_2.csv`  | 1000000 |    10 | 2011-12-31 | 2012-05-29 | 219791 | 98.7% | 99.6%  | 99.7%  |
| `FY2012_3.csv`  | 1000000 |    10 | 2011-10-07 | 2011-12-31 | 180489 | 99.1% | 99.7%  | 99.8%  |
| `FY2012_4.csv`  |  274924 |    10 | 2011-10-01 | 2011-10-07 |  15074 | 99.0% | 100.0% | 100.0% |
| `FY2013_1.csv`  | 1000000 |    10 | 2013-06-07 | 2013-09-30 | 228070 | 97.8% | 100.0% | 99.9%  |
| `FY2013_2.csv`  | 1000000 |    10 | 2013-01-29 | 2013-06-07 | 230441 | 98.6% | 100.0% | 99.6%  |
| `FY2013_3.csv`  | 1000000 |    10 | 2012-10-05 | 2013-01-29 | 194301 | 99.0% | 100.0% | 98.5%  |
| `FY2013_4.csv`  |  436000 |    10 | 2012-10-01 | 2012-10-05 |   3152 | 99.0% | 100.0% | 100.0% |
| `FY2014_1.csv`  | 1000000 |    10 | 2014-06-27 | 2014-09-30 | 202536 | 97.8% | 100.0% | 99.9%  |
| `FY2014_2.csv`  | 1000000 |    10 | 2014-02-28 | 2014-06-27 | 235792 | 98.1% | 100.0% | 99.9%  |
| `FY2014_3.csv`  | 1000000 |    10 | 2013-10-31 | 2014-02-28 | 225858 | 98.7% | 100.0% | 100.0% |
| `FY2014_4.csv`  |  818183 |    10 | 2013-10-01 | 2013-10-31 |  49032 | 99.0% | 100.0% | 100.0% |
| `FY2015_1.csv`  | 1000000 |    10 | 2015-06-17 | 2015-09-30 | 207919 | 97.4% | 100.0% | 99.9%  |
| `FY2015_2.csv`  | 1000000 |    10 | 2015-02-10 | 2015-06-17 | 228750 | 98.2% | 100.0% | 99.9%  |
| `FY2015_3.csv`  | 1000000 |    10 | 2014-10-28 | 2015-02-10 | 211778 | 98.6% | 100.0% | 100.0% |
| `FY2015_4.csv`  |  328425 |    10 | 2014-10-01 | 2014-10-28 |  44285 | 98.9% | 100.0% | 100.0% |
| `FY2016_1.csv`  | 1000000 |    10 | 2016-06-28 | 2016-09-30 | 193095 | 97.8% | 100.0% | 99.9%  |
| `FY2016_2.csv`  | 1000000 |    10 | 2016-02-25 | 2016-06-28 | 227867 | 98.1% | 100.0% | 99.9%  |
| `FY2016_3.csv`  | 1000000 |    10 | 2015-11-03 | 2016-02-25 | 201722 | 98.2% | 100.0% | 100.0% |
| `FY2016_4.csv`  |  632643 |    10 | 2015-10-01 | 2015-11-03 |  50961 | 99.0% | 100.0% | 100.0% |
| `FY2017_1.csv`  | 1000000 |    10 | 2017-06-30 | 2017-09-30 | 146121 | 99.4% | 100.0% | 100.0% |
| `FY2017_2.csv`  | 1000000 |    10 | 2017-03-23 | 2017-06-30 | 174485 | 99.5% | 100.0% | 100.0% |
| `FY2017_3.csv`  | 1000000 |    10 | 2016-12-05 | 2017-03-23 | 192208 | 99.3% | 100.0% | 100.0% |
| `FY2017_4.csv`  |  887842 |    10 | 2016-10-01 | 2016-12-05 | 137418 | 99.0% | 100.0% | 100.0% |
| `FY2018_1.csv`  | 1000000 |    10 | 2017-10-01 | 2018-09-30 |  85505 | 99.6% | 100.0% | 100.0% |
| `FY2018_2.csv`  | 1000000 |    10 | 2017-10-01 | 2018-09-30 |  88338 | 99.6% | 100.0% | 100.0% |
| `FY2018_3.csv`  | 1000000 |    10 | 2017-10-01 | 2018-09-30 |  85913 | 99.6% | 100.0% | 100.0% |
| `FY2018_4.csv`  | 1000000 |    10 | 2017-10-01 | 2018-09-30 |  87780 | 99.5% | 100.0% | 100.0% |
| `FY2018_5.csv`  | 1000000 |    10 | 2017-10-01 | 2018-09-30 |  88827 | 99.5% | 100.0% | 100.0% |
| `FY2018_6.csv`  | 1000000 |    10 | 2017-10-01 | 2018-09-30 |  85332 | 99.5% | 100.0% | 100.0% |
| `FY2018_7.csv`  | 1000000 |    10 | 2017-10-01 | 2018-09-30 |  87710 | 99.6% | 100.0% | 100.0% |
| `FY2018_8.csv`  |  263073 |    10 | 2017-10-01 | 2018-09-30 |  24243 | 99.5% | 100.0% | 100.0% |
| `FY2019_1.csv`  | 1000000 |    10 | 2018-10-01 | 2019-09-30 | 178908 | 99.6% | 100.0% | 100.0% |
| `FY2019_2.csv`  | 1000000 |    10 | 2018-10-01 | 2019-09-30 | 181622 | 99.6% | 100.0% | 100.0% |
| `FY2019_3.csv`  | 1000000 |    10 | 2018-10-01 | 2019-09-30 | 183549 | 99.6% | 100.0% | 100.0% |
| `FY2019_4.csv`  | 1000000 |    10 | 2018-10-01 | 2019-09-30 | 182285 | 99.6% | 100.0% | 100.0% |
| `FY2019_5.csv`  | 1000000 |    10 | 2018-10-01 | 2019-09-30 | 182443 | 99.6% | 100.0% | 100.0% |
| `FY2019_6.csv`  | 1000000 |    10 | 2018-10-01 | 2019-09-30 | 182384 | 99.6% | 100.0% | 100.0% |
| `FY2019_7.csv`  | 1000000 |    10 | 2018-10-01 | 2019-09-30 | 181664 | 99.6% | 100.0% | 100.0% |
| `FY2019_8.csv`  |  843036 |    10 | 2018-10-01 | 2019-09-30 | 153240 | 99.6% | 100.0% | 100.0% |
| `FY2020_1.csv`  | 1000000 |    10 | 2019-12-02 | 2020-09-30 | 415899 | 99.7% | 100.0% | 100.0% |
| `FY2020_2.csv`  | 1000000 |    10 | 2019-10-17 | 2019-12-02 | 226640 | 99.6% | 100.0% | 100.0% |
| `FY2020_3.csv`  | 1000000 |    10 | 2019-10-08 | 2019-10-17 |  50671 | 99.5% | 100.0% | 100.0% |
| `FY2020_4.csv`  | 1000000 |     9 | 2019-10-04 | 2019-10-08 |  13856 | 99.4% | 100.0% | 100.0% |
| `FY2020_5.csv`  |   74967 |     9 | 2019-10-01 | 2019-10-04 |  20449 | 99.5% | 100.0% | 100.0% |
| `FY(All)_1.csv` |  985905 |    10 | 2007-07-13 | 2020-09-30 | 407000 | 99.7% | 100.0% | 100.0% |
| `FY(All)_1.csv` |  985905 |    10 | 2007-07-13 | 2020-09-30 | 407000 | 99.7% | 100.0% | 100.0% |
