---
output:
  pdf_document: default
urlcolor: blue
header-includes:    
  - \usepackage{lastpage}
  - \usepackage{fancyhdr}
  - \pagestyle{fancy}
  - \fancyhead[CO, CE]{Claire Hsiung, 1004189736}
  - \fancyfoot[CO, CE]{\thepage \ of \pageref{LastPage}}
---

```
# Students: You probably shouldn't change any of the code in this chunk.

# These are the packages you will need for this activity
packages_needed <- c("tidyverse", "googledrive", "readxl", "janitor", 
                     "lubridate", "opendatatoronto", "ggthemes")

package.check <- lapply(
  packages_needed,
  FUN = function(x) {
    if (!require(x, character.only = TRUE)) {
      install.packages(x, dependencies = TRUE)
    }
  }
)

# Credit: package.check based on a helpful post from Vikram Baliga
#https://vbaliga.github.io/verify-that-r-packages-are-installed-and-loaded/

# Load tidyverse
library(tidyverse)
library(readxl)
library(janitor)
library(opendatatoronto)
library(ggthemes)

# Set so that long lines in R will be wrapped:
knitr::opts_chunk$set(tidy.opts=list(width.cutoff=80), echo = TRUE)
```


```
# Students: You probably shouldn't change any of the code in this chunk BUT...

# This chunk loads the most recent data from Toronto City and the data from OpenToronto.

# You have to RUN this chunk by hand to update the data as 
#   eval is set to FALSE to limit unnecessary requsts on the site.

###################################################
# Step one: Get the COVID data from Toronto City. #
###################################################

googledrive::drive_deauth()

url1 <- "https://drive.google.com/file/d/11KF1DuN5tntugNc10ogQDzFnW05ruzLH/view"
googledrive::drive_download(url1, path="data/CityofToronto_COVID-19_Daily_Public_Reporting.xlsx", overwrite = TRUE)

url2 <- "https://drive.google.com/file/d/1jzH64LvFQ-UsDibXO0MOtvjbL2CvnV3N/view"
googledrive::drive_download(url2, path = "data/CityofToronto_COVID-19_NeighbourhoodData.xlsx", overwrite = TRUE)

# this removes the url object that we don't need anymore
rm(url1, url2)

#####################################################################
# Step two: Get the data neighbourhood data from Open Data Toronto. #
#####################################################################

nbhoods_shape_raw <- list_package_resources("neighbourhoods") %>% 
  get_resource()

saveRDS(nbhoods_shape_raw, "data/neighbourhood_shapefile.Rds")

nbhood_profile <- search_packages("Neighbourhood Profile") %>%
  list_package_resources() %>% 
  filter(name == "neighbourhood-profiles-2016-csv") %>% 
  get_resource()

saveRDS(nbhood_profile, "data/neighbourhood_profile.Rds")
```


```
######################################################
# Step three: Load the COVID data from Toronto City. #
######################################################

# Saving the name of the file as an object and then using the object name in the
# following code is a helpful practice. Why? If we change the name of the file 
# being used, we'll only have to change it in one place. This helps us avoid 
# 'human error'.

daily_data <- "data/CityofToronto_COVID-19_Daily_Public_Reporting.xlsx"

# Cases reported by date (double check the sheet is correct)
# Should be a sheet names something like  
## 'Cases by Reported Date'
reported_raw <- read_excel(daily_data, sheet = 5) %>% 
  clean_names()

# Cases by outbreak type (double check the sheet is correct)
# Should be a sheet names something like  
## 'Cases by Outbreak Type and Epis'
outbreak_raw <- read_excel(daily_data, sheet = 3) %>% 
  clean_names()

# When was this data updated?
date_daily <- read_excel(daily_data, sheet = 1) %>% 
  clean_names()

# By neighbourhood
neighbourood_data <- "data/CityofToronto_COVID-19_NeighbourhoodData.xlsx"

# Cases reported by date
nbhood_raw <- read_excel(neighbourood_data, sheet = 2) %>% 
  clean_names()

# Date the neighbourhood data was last updated
date_nbhood <- read_excel(neighbourood_data, sheet = 1) %>% 
  clean_names()

#don't need these anymore
rm(daily_data, neighbourood_data)

#############################################################
# Step four: Load the neighbourhood data from Toronto City. #
#############################################################

# Get neighbourhood profile data
nbhood_profile <- readRDS("data/neighbourhood_profile.Rds")

# Get shape data for mapping 
nbhoods_shape_raw <- readRDS("data/neighbourhood_shapefile.Rds") %>% 
  sf::st_as_sf() ## Makes sure shape info is in the most up to date format

```

Code last run `r Sys.Date()`.  
Daily: `r date_daily[1,1]`.   
Neighbourhood: `r date_nbhood[1,1]`. 

# Task 1: Daily cases
## Data wrangling


```
reported <- reported_raw %>%
 mutate_if(is.numeric, replace_na, replace = 0) %>% 
mutate(reported_date=as.Date(reported_date, format = "%d.%b.%Y")) %>% 
  rename(Active = active,
         Recovered = recovered,
         Deceased = deceased) %>% 
  pivot_longer(-c(reported_date), names_to = "Status", values_to = "Cases") %>%
  mutate(Status = fct_relevel(Status, "Recovered", after = 1))

```

\newpage
## Data visualization


```

reported %>% ggplot(aes(x = reported_date, y = Cases, fill = Status)) + 
  geom_bar(stat = "identity") +
  scale_x_date(limits = as.Date(c('2020-01-01', Sys.Date())), 
               labels = scales::date_format("%d %b %y"))+
  theme_minimal()+
  labs(title = "Cases reported by day in Toronto, Canada", 
       subtitle = "Confirmed and probable cases",
       x = "Date",
       y = "Case count",
       caption = str_c("Created by: Claire Hsiung for STA303/1002, U of T \n
       Source: Ontario Ministry of Health, Integrated Public Health Information System and CORES \n" ,
       date_daily[1,1], sep = " ")) + 
  theme(legend.title = element_blank(), legend.position = c(0.15, 0.8)) + 
  scale_y_continuous(limits = c(0,2000)) + 
  scale_fill_manual(values = c("#003F5C","#86BCB6","#B9CA5D")) 
```

\newpage
# Task 2: Outbreak type
## Data wrangling



```
outbreak <- outbreak_raw %>% mutate(episode_week = date(episode_week)) %>%
  mutate(outbreak_or_sporadic = case_when(
    outbreak_or_sporadic == "OB Associated" ~ "Outbreak associated",
    outbreak_or_sporadic == "Sporadic" ~ "Sporadic"
  )) %>% 
  group_by(episode_week) %>%
mutate(total_cases = sum(cases)) %>%
  mutate(outbreak_or_sporadic = fct_relevel(outbreak_or_sporadic, "Sporadic", after = 1))
  
```

\newpage
## Data visualization


```
 
outbreak %>% ggplot(aes(x = episode_week, y = cases, fill = outbreak_or_sporadic)) + 
  geom_bar(stat = "identity") +
  scale_x_date(limits = as.Date(c('2020-01-01', Sys.Date())),
               labels = scales::date_format("%d %b %y"))+
  theme_minimal()+
  labs(title = "Cases by outbreak type and week in Toronto, Canada", 
       subtitle = "Confirmed and probable cases",
       x = "Date",
       y = "Case count",
       caption = str_c("Created by: Claire Hsiung for STA303/1002, U of T \n
       Source: Ontario Ministry of Health, Integrated Public Health Information System and CORES \n" , 
       date_daily[1,1], sep = " ")) + 
  theme(legend.title = element_blank(), 
        legend.position = c(0.15, 0.8)) + 
  scale_y_continuous(limits = c(0,7000)) + 
  scale_fill_manual(values = c("#86BCB6","#B9CA5D")) 

```

\newpage
# Task 3: Neighbourhoods
## Data wrangling: part 1


```
income <- nbhood_profile %>% 
  filter(`_id` == 1143) %>% 
  pivot_longer(-c("_id", "Category", "Topic", "Data Source", "Characteristic"), 
               names_to = "neighbourhood_name", 
               values_to = "Percentage") %>% 
  mutate_at('Percentage', parse_number)



```

## Data wrangling: part 2


```
nbhoods_all<- nbhoods_shape_raw %>% 
  mutate(neighbourhood_name = str_remove(AREA_NAME, "\\s\\(\\d+\\)")) %>%
  full_join(income, by = "neighbourhood_name") %>% 
  full_join(nbhood_raw, by = "neighbourhood_name") %>%
  rename(rate_per_100000 = rate_per_100_000_people)

problems <- nbhoods_all %>%
  filter(is.na(neighbourhood_id))

#updating problematic rows in income using information in problems
income <- income[-1,]
nbhoods_shape_raw <- nbhoods_shape_raw %>% 
  mutate(AREA_NAME = str_replace(AREA_NAME, "Cabbagetown-South St.James Town", 
                                 "Cabbagetown-South St. James Town")) %>%  
  mutate(AREA_NAME = str_replace(AREA_NAME, "North St.James Town", 
                                 "North St. James Town")) %>% 
  mutate(AREA_NAME = str_replace(AREA_NAME, "Weston-Pellam Park", 
                                 "Weston-Pelham Park")) 

nbhoods_all <- nbhoods_shape_raw %>% 
  mutate(neighbourhood_name = str_remove(AREA_NAME, "\\s\\(\\d+\\)")) %>%
  full_join(income, by = "neighbourhood_name") %>% 
  full_join(nbhood_raw, by = "neighbourhood_name") %>%
  rename(rate_per_100000 = rate_per_100_000_people)%>%
  drop_na(AREA_SHORT_CODE)


```

## Data wrangling: part 3


```
med_inc = median(nbhoods_all$Percentage)
med_rate = median(nbhoods_all$rate_per_100000) 
nbhoods_final <- nbhoods_all  %>% 
  mutate(nbhood_type = case_when(
    (Percentage >= med_inc & rate_per_100000 >= med_rate)
    ~ "Higher low income rate, higher case rate",
    (Percentage >= med_inc & rate_per_100000 <= med_rate)
    ~ "Higher low income rate, lower case rate",
    (Percentage <= med_inc & rate_per_100000 >= med_rate) 
    ~ "Lower low income rate, higher case rate",
    (Percentage <= med_inc & rate_per_100000 <= med_rate)
    ~ "Lower low income rate, lower case rate"
  ))

```

\newpage
## Data visualization


```
ggplot(data = nbhoods_final) +
geom_sf(aes(geometry = geometry, fill = Percentage)) +
theme_map() + scale_fill_gradient(name=
"% low income", low = "darkgreen", high = "lightgrey")+
  labs(title = "Percentage of 18 to 64 year olds living in a low income family (2015)", 
       subtitle = "Neighbourhoods in Toronto, Canada",
  caption = str_c("Created by: Claire Hsiung for STA303/1002, U of T \n
       Source: Census Profile 98-316-X2016001 via OpenData Toronto \n" ,
       date_daily[1,1], sep = " ")) +
  theme(legend.position = "right")

 
```

\newpage


```
ggplot(data = nbhoods_final) +
geom_sf(aes(geometry = geometry, fill = rate_per_100000)) +
theme_map() + scale_fill_gradient(name=
"Cases per 100,000 people", low = "white", high = "darkorange")+
  labs(title = "COVID-19 cases per 100,000, by neighbourhood in Toronto, Canada",
  caption = str_c("Created by: Claire Hsiung for STA303/1002, U of T \n
       Source: Ontario Ministry of Health, Integrated Public Health Information System and CORES \n" , 
       date_daily[1,1], sep = " ")) +
  theme(legend.position = "right")


```

\newpage


```
ggplot(data = nbhoods_final) +
geom_sf(aes(geometry = geometry, fill = nbhood_type)) +
theme_map() + scale_fill_brewer(palette = "Set1") + 
  labs(fill = "% of 18 to 64 year-olds in low income families and 
       COVID-19 case rates", 
       title = "COVID-19 cases per 100,000, by neighbourhood in Toronto, Canada",
  caption = str_c("Created by: Claire Hsiung for STA303/1002, U of T \n
  Income data source: Census Profile 98-316-X2016001 via OpenData Toronto \n
       COVID data source: Ontario Ministry of Health,
  Integrated Public Health Information System and CORES \n" , 
       date_daily[1,1], sep = " ")) +
  theme(legend.position = "right")

```






```
# This chunk of code helps you prepare your assessment for submission on Crowdmark
# This is optional. If it isn't working, you can do it manually/take another approach.

# Run this chunk by hand after knitting your final version of your pdf for submission.
# A new file called 'to_submit' will appear in your working directory with each page of your assignment as a separate pdf.

# Install the required packages
if(!match("staplr", installed.packages()[,1], nomatch = FALSE))
  {install.packages("staplr")}

# Don't edit anything in this function
prep_for_crowdmark <- function(pdf=NULL){
  # Get the name of the file you're currently in. 
  this_file <- rstudioapi::getSourceEditorContext()$path
  pdf_name <- sub(".Rmd", ".pdf", sub('.*/', '', this_file))
  
  # Create a file called to_submit to put the individual files in
  # This will be in the same folder as this file is saved
  if(!match("to_submit", list.files(), nomatch = FALSE))
    {dir.create("to_submit")}
 
  # Split the files
  if(is.null(pdf)){
  staplr::split_pdf(pdf_name, output_directory = "to_submit", prefix = "page_")} else {
    staplr::split_pdf(pdf, output_directory = "to_submit", prefix = "page_") 
  }
}

prep_for_crowdmark()

```
