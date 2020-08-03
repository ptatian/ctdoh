Households by size
================

``` r
library(tidyverse)
library(tidycensus)
library(janitor)
library(cwi)
library(camiller)
```

Same deal as with households by type… this number moves only very
slightly, so I’ll just pull three points: 2000, 2010, and 2018

HH size by tenure… B08201 (2010, 2018) 2000: H015 sf1

Groupings go from 1 to 7+ but I’ll regroup to 1 to 4+

# Fetch

``` r
years <- list("2010" = 2010, "2018" = 2018)
fetch1018 <- years %>% map(~multi_geo_acs(table = "B25009", year = ., new_england = F)) %>%
    bind_rows() %>% 
    label_acs() %>% 
    clean_names() %>% 
    select(-moe) %>% 
    rename(value = estimate)

fetch00 <- multi_geo_decennial(table = "H015", year = 2000) %>% 
    label_decennial(year = 2000) %>% 
    clean_names()
```

# Clean

``` r
size1018 <- fetch1018 %>% 
    select(-geoid, -state, -variable) %>% 
    group_by(level, name, year) %>% 
    add_grps(list(total = 1,
                                owner_total = 2,
                                owner_p1 = 3, 
                                owner_p2 = 4,
                                owner_p3 = 5,
                                owner_p4_or_more = c(6:9),
                                renter_total = 10,
                                renter_p1 = 11, 
                                renter_p2 = 12,
                                renter_p3 = 13,
                                renter_p4_or_more = c(14:17)),
                     value = value, group = label)

size <- fetch00 %>% 
    select(-geoid, -state, -variable) %>%
    group_by(level, name, year) %>%
    add_grps(list(total = 1,
                                owner_total = 2,
                                owner_p1 = 3, 
                                owner_p2 = 4,
                                owner_p3 = 5,
                                owner_p4_or_more = c(6:9),
                                renter_total = 10,
                                renter_p1 = 11, 
                                renter_p2 = 12,
                                renter_p3 = 13,
                                renter_p4_or_more = c(14:17)),
                     value = value, group = label) %>% 
    bind_rows(size1018) %>% 
    rename(hhlds = label) %>% 
    mutate(tenure = if_else(grepl("renter", hhlds), "renter", "owner"),
                 tenure = if_else(hhlds == "total", "total", tenure)) %>% 
    mutate(hhlds = str_remove(hhlds, "owner_"),
                 hhlds = str_remove(hhlds, "renter_")) %>% 
    select(level, name, year, tenure, hhlds, value)

write_csv(size, "../output_data/household_size_by_tenure_2000_2018.csv")

renter <- size %>% 
    filter(tenure == "renter") %>% 
    ungroup() %>% 
    group_by(level, name, year, tenure) %>% 
    calc_shares(group = hhlds, denom = "total", value = value)

write_csv(renter, "../output_data/household_size_renter_2000_2018.csv")

owner <- size %>% 
    filter(tenure == "owner") %>% 
    ungroup() %>% 
    group_by(level, name, year, tenure) %>%  
    calc_shares(group = hhlds, denom = "total", value = value)

write_csv(owner, "../output_data/household_size_owner_2000_2018.csv")
```

``` r
bind_rows(owner, renter) %>% 
    mutate(year = as.factor(year)) %>% 
    filter(name == "Connecticut") %>% 
    ggplot(aes(year, share, group = tenure)) +
    geom_col(aes(fill = hhlds)) +
    facet_grid(facets = "tenure") +
    theme(axis.text.x = element_text(angle = -90)) +
    labs(title = "CT households")
```

![](hh_by_size_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

# Calculate change

``` r
household_size_change <- size %>%
    group_by(level, hhlds, tenure) %>%
    arrange(name, year, hhlds) %>%
    mutate(diff = value - lag(value, default = first(value))) %>%
    arrange(level, year, hhlds, tenure) %>%
    rename(change_from_prev_data_year = diff)

household_size_change %>% 
    write_csv("../output_data/household_size_change_2000_2018.csv")
```