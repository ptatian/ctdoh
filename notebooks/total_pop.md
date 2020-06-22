Total population trends
================

``` r
library(tidyverse)
library(tidycensus)
library(janitor)
library(cwi)
library(camiller)
```

Collecting and lightly cleaning basic population data for multiple
geographies and each year available starting in 2000 through latest
available.

# Total pop

## Fetch

State and county: Pre-2010 state and county data from Census intercensal
counts. 2010-2018 data from deci/ACS.

Town: only 2011-2018 ACS, 2000 and 2010 deci.

``` r
# 2000-2010 intercensal data comes in spreadsheets from
# https://www.census.gov/data/datasets/time-series/demo/popest/intercensal-2000-2010-counties.html
# see data dictionary in companion pdf
intercensal <- read_csv("../input_data/co-est00int-alldata-09.csv") %>%
    clean_names()

acs_years <- list("2011" = 2011, "2012" = 2012, "2013" = 2013, "2014" = 2014, "2015" = 2015, "2016" = 2016, "2017" = 2017, "2018" = 2018)

#b01001 - sex by age
sex_by_age <- acs_years %>% map(~multi_geo_acs(table = "B01001", year = ., new_england = F))
sex_by_age_bind <- Reduce(rbind, sex_by_age) %>% label_acs()


deci_years <- list("2000" = 2000, "2010" = 2010)

deci_pops <- deci_years %>% map(~multi_geo_decennial(table = "P001", year = .))
deci_pops_bind <- Reduce(rbind, deci_pops) %>% label_acs() 
```

## Clean

``` r
deci_pop <- deci_pops_bind %>%
    mutate(var = "total_pop",  moe = 0) %>% 
    select(year, level, geoid = GEOID, name = NAME, county, var, estimate = value, moe)

acs_pop <- sex_by_age_bind %>%
    separate(label, into = c("total", "gender", "age"), sep = "!!", fill = "right") %>% 
    clean_names() %>% 
    filter(grepl("_001", variable)) %>% 
    mutate(var = "total_pop",
                 var = as.factor(var)) %>% 
    mutate(moe = replace_na(moe, 0)) %>% 
    select(year, level, geoid, name, county, var, estimate, moe)

period_lut <- tibble(
    estimate_date = c(
        "remove_april_2000",
        "remove_july_2000",
        seq(2001, 2009),
        "remove_april_2010",
        "remove_july_2010"),
    year = seq(1:13))

int_county <- intercensal %>% 
    mutate(geoid = paste(state, county, sep = "")) %>% 
  select(geoid, name = ctyname, year, agegrp, estimate = tot_pop) %>% 
    left_join(period_lut, by = "year") %>% 
    filter(agegrp == "99", !grepl("remove", estimate_date)) %>% 
    mutate(year2 = as.numeric(estimate_date),
                 moe = 0) %>% 
    mutate(var = "total_pop", level = "2_counties", county = NA, moe = 0,
                 var = as.factor(var)) %>% 
    select(year = year2, level, geoid, name, county, var, estimate, moe)

int_ct <- int_county %>% 
    select(-level, -geoid, -name) %>% 
    group_by(year, county, var) %>% 
    summarise(estimate = sum(estimate), moe = sum(moe)) %>% 
    ungroup() %>% 
    mutate(level = "1_state", geoid = "09", name = "Connecticut") %>% 
    select(year, level, geoid, name, county, var, estimate, moe)

int_pop <- bind_rows(int_ct, int_county) %>% 
    mutate(level = as.factor(level))

### write out total pop
pop_out <- bind_rows(deci_pop, int_pop, acs_pop) %>% 
    arrange(level, geoid, year) %>% 
    write_csv(., "../output_data/total_pop_2000_2018.csv")
```

## Calculate pop change

``` r
pop_change <- pop_out %>% 
    group_by(level, geoid, county, var) %>% 
    arrange(name, year) %>% 
    mutate(diff = estimate - lag(estimate, default = first(estimate))) %>% 
    arrange(level, geoid, year) %>% 
    mutate(var = "pop_change") %>% 
    select(-estimate, -moe) %>% 
    rename(estimate = diff) %>% 
    select(year, level, geoid, name, county, var, estimate) %>% 
    write_csv("../output_data/pop_change_2000_2018.csv")
```