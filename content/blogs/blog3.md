---
categories:
- ""
- ""
date: "2017-10-31T22:26:13-05:00"
description: ""
draft: false
image: database.jpg
keywords: ""
slug: hw3
title: Databases and Web Scraping
---

```{r}
#| label: load-libraries
#| echo: false # This option disables the printing of code (only output is displayed).
#| message: false
#| warning: false

library(tidyverse)
library(wbstats)
library(tictoc)
library(skimr)
library(countrycode)
library(here)
library(DBI)
library(dbplyr)
library(arrow)
library(rvest)
library(robotstxt) # check if we're allowed to scrape the data
library(scales)
library(sf)
library(readxl)
library(stringr)
```

# Money in UK politics

[The Westminster Accounts](https://news.sky.com/story/the-westminster-accounts-12786091), a recent collaboration between Sky News and Tortoise Media, examines the flow of money through UK politics. It does so by combining data from three key sources:

1.  [Register of Members' Financial Interests](https://www.parliament.uk/mps-lords-and-offices/standards-and-financial-interests/parliamentary-commissioner-for-standards/registers-of-interests/register-of-members-financial-interests/),
2.  [Electoral Commission records of donations to parties](http://search.electoralcommission.org.uk/English/Search/Donations), and
3.  [Register of All-Party Parliamentary Groups](https://www.parliament.uk/mps-lords-and-offices/standards-and-financial-interests/parliamentary-commissioner-for-standards/registers-of-interests/register-of-all-party-party-parliamentary-groups/).

You can [search and explore the results](https://news.sky.com/story/westminster-accounts-search-for-your-mp-or-enter-your-full-postcode-12771627) through the collaboration's interactive database. Simon Willison [has extracted a database](https://til.simonwillison.net/shot-scraper/scraping-flourish) and this is what we will be working with. If you want to read more about [the project's methodology](https://www.tortoisemedia.com/2023/01/08/the-westminster-accounts-methodology/).

## Open a connection to the database

The database made available by Simon Willison is an `SQLite` database

```{r}
sky_westminster <- DBI::dbConnect(
  drv = RSQLite::SQLite(),
  dbname = here::here("data", "sky-westminster-files.db")
)
```

How many tables does the database have?

```{r}

#Identify the number of tables in the remote database
num_tables <- length(DBI::dbListTables(sky_westminster))

```

**Answer:** The are 7 tables in the sky_westminster database.

## Which MP has received the most amount of money?

```{r}

#Browse tables in the database
DBI::dbListTables(sky_westminster)

#Store the 'payments' table as a database object
payments_db <- dplyr::tbl(sky_westminster, "payments")

#Glimpse payments_db to check for contents
glimpse(payments_db)

#Store the 'members' table as a database object
members_db <- dplyr::tbl(sky_westminster, "members")

#Glimpse members_db to check for contents
glimpse(members_db)

#Compute for the amounts that each MP received
mp_amount_summary <- payments_db %>% 
  
#Group by member id
  group_by(member_id) %>%
  
#Compute for the sum of donations for each member id
  summarize(total_value = sum(value)) %>% 
  
#Arrange in descending order
  arrange(desc(total_value))

#Extract the member ID of the MP with the most amount
mp_with_most_money <- mp_amount_summary %>% 
  head(1)

#Identify the name of the MP using left_join
mp_name <- left_join(mp_with_most_money, members_db, by = c("member_id" = "id")) %>% 
  collect()

```

**Answer:** Theresa May was the MP who has received the most amount of money with 2809765.

## Any `entity` that accounts for more than 5% of all donations?

Is there any `entity` whose donations account for more than 5% of the total payments given to MPs over the 2020-2022 interval? Who are they and who did they give money to?

```{r}

#Identify sum of total payments made to MPs and store into a variable
total_payments <- payments_db %>% 
  summarize(total_payments = sum(value)) %>% 
  
#Extract only the value for total payments
  pull(total_payments)

#Compute for donations made by each entity and store into a variable
entity_donations <- payments_db %>% 
  
#Group by entity name and member id of the MP
  group_by(entity, member_id) %>% 

#Compute for the total value of payments for each entity
  summarize(total_donations = sum(value)) %>% 
  
#Add a new column for the percentage of the donation for each entity over the total payments
  mutate(percentage = (total_donations / total_payments) * 100, .groups = 'drop') %>% 
  
#Ungroup data
  ungroup() %>%
  
#Use left join to combine with the members_db dataset
  left_join(collect(members_db), by = c("member_id" = "id"), copy = TRUE)

#Identify entities that have made the highest contributions and store into a variable
high_donation_entities <- entity_donations %>%
  
#Filter for the entities that made more than 5% in contributions
  filter(percentage > 5) %>% 
  collect()

```

**Answer:** Withers LLP made more than 5% in donations or 1812732 in nominal amount to Sir Geoffrey Cox over the 2020-2022 period.

## Do `entity` donors give to a single party or not?

-   How many distinct entities who paid money to MPs are there?
-   How many (as a number and %) donated to MPs belonging to a single party only?

```{r}

#Store the 'parties' table as a database object
parties_db <- dplyr::tbl(sky_westminster, "parties")

#Store the 'party_donations' table as a database object
party_donations_db <- dplyr::tbl(sky_westminster, "party_donations")

#Determine the number of distinct entities and store into a variable
num_entities <- party_donations_db %>% 
  
#Identify the distinct entities
  distinct(entity) %>% 
  
#Count the number of distinct entities
  summarize(num_entities = n()) %>% 
  
#Extract the value of the number of entities
  pull(num_entities)


#Identify the entities that donated to a single party and store into a variable
entities_single_party <- party_donations_db %>% 
  
#Group by entity
  group_by(entity) %>%
  
#Compute for the number of distinct party id
  summarize(num_parties = n_distinct(party_id)) %>% 
  
#Filter to obtain the entity that donated to a single party
  filter(num_parties == 1)

#Identify the number of entities that donated to a single party and store into a variable
num_entites_single_party <- entities_single_party %>%
  
#Compute for the number of these entities
  summarize(num_entites_single_party = n()) %>% 
  
#Extract only the number of these entities
  pull(num_entites_single_party)

#Compute for the percentage and store into a variable
percentage_entities_single_party <- (num_entites_single_party/num_entities)*100

```

**Answer:** There are 1077 distinct entities that donated money to MPs. Out of these entities, 1068 (99%) donated to MPs belonging to a single party.

## Which party has raised the greatest amount of money in each of the years 2020-2022?

```{r}

#Compute for the total amounts raised per year by each party and store into a variable
party_year_amounts <- party_donations_db %>%
  
#Group by year and party id
  group_by(year = substr(date, 1, 4), party_id) %>%
  
#Compute for the sum for each party per year
  summarise(total_amount = sum(value), .groups = "drop")

#Identify the party with the highest amount per year and store into a variable
max_party_per_year <- party_year_amounts %>%
  
#Group by year
  group_by(year) %>%
  
#Filter for the maximum amount
  filter(total_amount == max(total_amount)) %>%
  
#Use inner join to combine with the parties_db dataset
  inner_join(parties_db, by = c("party_id" = "id"), copy = TRUE) %>% 
  
#Select relevant columns
  select(year, party_name = name, total_amount) %>%
  
#Collect the data
  collect()

```

**Answer:** For each of the years from 2020 to 2022, the Conservative party has raised the most amount of money: 42770782 for 2020, 17718212 for 2021, and 15568476 for 2022.

I would like you to write code that generates the following table.

```{r echo=FALSE, out.width="80%"}
knitr::include_graphics(here::here("images", "total_donations_table.png"), error = FALSE)
```

```{r}

#Use inner join to combine the party_donations_db and parties_db datasets
result <- party_donations_db %>%
  inner_join(parties_db, by = c("party_id" = "id")) %>%
  
#Group by year and party name
  group_by(year = substr(date, 1, 4), name) %>%
  
#Compute for the total donations per year for each party
  summarise(total_year_donations = sum(value)) 

#Calculate the total year donations for each year and store into a variable
total_year_donations <- result %>%
  
#Group by year
  group_by(year) %>%
  
#Compute for the total donations per year and convert to numeric
  summarise(total_donations = sum(as.numeric(total_year_donations)))

#Use inner join to combine data with total yearly donations by year
result <- result %>%
  inner_join(total_year_donations, by = "year", copy = TRUE) %>%
  
#Add another column for the proportion of total donations per party by the total donations per year
  mutate(
    prop = as.numeric(total_year_donations) / total_donations
  ) %>%
  
#Ungroup the data
  ungroup() %>%
  
#Remove the total donations column
  select(-total_donations) %>% 
  
#Collect the data
  collect()

```

... and then, based on this data, plot the following graph.

```{r echo=FALSE, out.width="80%"}
knitr::include_graphics(here::here("images", "total_donations_graph.png"), error = FALSE)
```

```{r}

#Arrange amount of total donations per year by party in descending order
result <- result %>%
  mutate(name = fct_reorder(name, total_year_donations, .desc = TRUE))

#Use ggplot to create graph
result %>% 
ggplot() +
  
#Set year as the x-axis, total donations per year by party as the y-axis, and the party name as the fill
  aes(x = year, y = total_year_donations, fill = name) +
  
#Create the bar graph and set position
  geom_col(position = "dodge") +
  
#Create labels for the graph
  labs(x = "", y = "", 
       title = "Conservatives have captured the majority of political donations",
       subtitle = "Donations to political parties, 2020-2022", fill = "Party") +
  
#Set theme
  theme_minimal() +
  
#Make the labels of the y-axis into integers
  scale_y_continuous(labels = scales::comma)

```

```{r}
dbDisconnect(sky_westminster)
```

# Anonymised Covid patient data from the CDC

```{r}
#| echo: false
#| message: false
#| warning: false


tic() # start timer
cdc_data <- open_dataset(here::here("data", "cdc-covid-geography"))
toc() # stop timer


glimpse(cdc_data)
```

Can you query the database and replicate the following plot?

```{r echo=FALSE, out.width="100%"}
knitr::include_graphics(here::here("images", "covid-CFR-ICU.png"), error = FALSE)
```

```{r}

#Calculate the number of deaths from the database and store into a variable
deaths_data <- cdc_data %>%
  
#Filter the data to select relevant columns within age group, sex, and ICU admission
  filter(sex %in% c("Male", "Female"), icu_yn %in% c("Yes", "No"), death_yn == "Yes") %>%
  select(age_group, sex, icu_yn)

#Group the number of deaths and store into a variable
grouped_deaths <- deaths_data %>%
  
#Group by sex, age group, and ICU admission
  group_by(sex, age_group, icu_yn) %>%
  
#Count the number of rows
  summarize(count = n()) %>%
  
#Collect the data
  collect()

#Calculate the number of cases from the database and store into a variable
cases_data <- cdc_data %>%
  
#Filter the data to select relevant columns within age group, sex, and ICU admission
  filter(sex %in% c("Male", "Female"), icu_yn %in% c("Yes", "No")) %>%
  select(age_group, sex, icu_yn)

#Group the number of deaths and store into a variable
grouped_cases <- cases_data %>%
  
#Group by sex, age group, and ICU admission
  group_by(sex, age_group, icu_yn) %>%
  
#Count the number of rows
  summarize(count = n()) %>%
  
#Collect the data
  collect()

#Calculate the CFR % and store into a variable
CFR_data <- grouped_deaths %>% 
  
#Use left join the combine with the grouped cases
  left_join(grouped_cases, by = c("sex", "age_group", "icu_yn")) %>% 
  
#Add another column to compute for the CFR %
  mutate(cfr = (count.x / count.y))

#Rename the data to ICU Admission/No ICU Admission
CFR_data <- CFR_data %>%
  mutate(icu_yn = recode(icu_yn, "Yes" = "ICU Admission", "No" = "No ICU Admission"))

#Plot the CFR % data using ggplot
CFR_data %>% 
ggplot() +
  
#Use the CFR % as the x-axis and age group as the y-axis
  aes(x = cfr, y = age_group) +
  
#Create the bar graph
  geom_bar(stat = "identity", fill = "#FF8F7C") +
  
#Include data labels in the graph and set desired positioning
  geom_text(aes(label = sprintf("%.0f", cfr * 100)), vjust = 0.5, hjust = 1, size = 2) +
  
#Facet grid by ICU Admission and sex
  facet_grid(icu_yn ~ sex, scales = "free", space = "free") +
  
#Remove axis labels and include title for the graph
  labs(x = "", y = "", title = "Covid CFR % by age group, sex and ICU Admission") +
  
#Set theme
  theme_bw() + 
  
#Set scale for the x-axis and title positioning
  scale_x_continuous(labels = scales::percent) +
  theme(plot.title = element_text(hjust = 0))


```

The previous plot is an aggregate plot for all three years of data. What if we wanted to plot Case Fatality Ratio (CFR) over time? Write code that collects the relevant data from the database and plots the following

```{r echo=FALSE, out.width="100%"}
knitr::include_graphics(here::here("images", "cfr-icu-overtime.png"), error = FALSE)
```

```{r}

#Calculate the number of deaths from the database and store into a variable
deaths_data1 <- cdc_data %>%
  
#Filter the data and select relevant columns within case month, age group, sex, and ICU admission
  filter(sex %in% c("Male", "Female"), icu_yn %in% c("Yes", "No"), death_yn == "Yes") %>%
  select(case_month, age_group, sex, icu_yn)

#Group the number of deaths and store into a variable
grouped_deaths1 <- deaths_data1 %>%
  
#Group by case month, sex, age group, and ICU admission
  group_by(case_month, sex, age_group, icu_yn) %>%
  
#Count the number of rows
  summarize(count = n()) %>%
  
#Collect the data
  collect()

#Calculate the number of cases and store into a variable
cases_data1 <- cdc_data %>%
  
#Filter the data and select relevant columns within case month, age group, sex, and ICU admission
  filter(sex %in% c("Male", "Female"), icu_yn %in% c("Yes", "No")) %>%
  select(case_month, age_group, sex, icu_yn)

#Group the number of cases and store into a variable
grouped_cases1 <- cases_data1 %>%
  
#Group by case month, sex, age group, and ICU admission
  group_by(case_month, sex, age_group, icu_yn) %>%
  
#Count the number of rows
  summarize(count = n()) %>%
  
#Collect the data
  collect()

#Calculate the CFR% and store into a variable
CFR_data1 <- grouped_deaths1 %>% 
  
#Use left join to combine with grouped cases
  left_join(grouped_cases1, by = c("case_month", "sex", "age_group", "icu_yn")) %>% 
  
#Add another column to compute for the CFR %
  mutate(cfr = (count.x / count.y))

#Rename the data to ICU Admission/No ICU Admission
CFR_data1 <- CFR_data1 %>%
  mutate(icu_yn = recode(icu_yn, "Yes" = "ICU Admission", "No" = "No ICU Admission"))

#Plot the CFR % data using ggplot
CFR_data1 %>% 
ggplot() +

#Set case month as the x-axis, CFR % as the y-axis, and age group as the grouping category
  aes(x = case_month, y = cfr, group = age_group, color = age_group) +
  
#Create the line graph
  geom_line() +
  
#Include data labels and set positioning
  geom_text(aes(label = sprintf("%.0f", cfr * 100)), vjust = -0.5, size = 3) +
  
#Use facet grid with ICU Admission and sex
  facet_grid(icu_yn ~ sex, scales = "free_y", space = "free") +
  
#Remove axis labels and include title for the graph
  labs(x = "", y = "", color = "Age Group", title = "Covid CFR % by age group, sex and ICU Admission") +
  
#Set theme
  theme_bw() +
  
#Adjust positioning of the case month and plot title, and remove the panel grids
  theme(axis.text.x = element_text(angle = 90, hjust = 1), plot.title = element_text(hjust = 0), panel.grid.major = element_blank(),panel.grid.minor = element_blank())

```

```{r}
urban_rural <- read_xlsx(here::here("data", "NCHSURCodes2013.xlsx")) %>% 
  janitor::clean_names() 
```

Can you query the database, extract the relevant information, and reproduce the following two graphs that look at the Case Fatality ratio (CFR) in different counties, according to their population?

```{r echo=FALSE, out.width="100%"}
knitr::include_graphics(here::here("images", "cfr-county-population.png"), error = FALSE)
```

```{r}

#Calculate the number of deaths from the database and store into a variable
deaths_data2 <- cdc_data %>%
  
#Filter for deaths
  filter(death_yn == "Yes") %>%
  
#Select relevant columns
  select(case_month, county_fips_code)

#Group the deaths and store into a variable
grouped_deaths2 <- deaths_data2 %>%
  
#Group by case month and county fips code
  group_by(case_month, county_fips_code) %>%
  
#Count the number of deaths per case month and county fips code
  summarize(count = n()) %>%
  
#Collect the data
  collect()

#Calculate the total case from the database and store into a variable
cases_data2 <- cdc_data %>%
  
#Select the relevant columns
  select(case_month, county_fips_code)

#Group the cases and store into a variable
grouped_cases2 <- cases_data2 %>%
  
#Group by case month and country fips code
  group_by(case_month, county_fips_code) %>%
  
#Count the number of cases by case month and county fips code
  summarize(count = n()) %>%
  
#Collect the data
  collect()

#Combine grouped deaths and cases into a single dataframe
CFR_data2 <- grouped_deaths2 %>% 
  
#Use left join to combined the data by case month and county fips code
  left_join(grouped_cases2, by = c("case_month", "county_fips_code")) %>% 
  
#Add new column to compute for the CFR %
  mutate(cfr = (count.x / count.y))

#Combine CFR data with the urban_rural dataframe
combined_data <- left_join(CFR_data2, urban_rural, by = c("county_fips_code" = "fips_code")) %>% 
  
#Filter out the NA values
  filter(!is.na(x2013_code)) 

#Group the combined data and store into a new variable
combined_data1 <- combined_data %>% 
  
#Group by case month, country fips code, and category name
  group_by(case_month, county_fips_code, x2013_code)%>% 
  
#Compute the CFR %
  summarize(cfr = sum(count.x) / sum(count.y))

#Use ggplot to graph the data
combined_data1 %>% 
  ggplot() +
  
#Set case month as the x-axis, CFR % as the y-axis, and the category name as the grouping as color
  aes(x = case_month, y = cfr, group = x2013_code, color = x2013_code) +
  
#Create the line graph
  geom_line() +
  
#Use facet wrap to create grids by category name and recode the category names
  facet_wrap(~ x2013_code, ncol = 2, scales = "free_y", labeller = labeller(x2013_code = c("1" = "1.Large central metro", "2" = "2.Large fringe metro", "3" = "3.Medium metro", "4" = "4.Small metropolitan", "5" = "5.Micropolitan", "6" = "6.Noncore"))) +
  
#Include labels for the graph
  labs(x = "", y = "", color = "x2013_code", title = "Covid CFR % by county population") +
  
#Include data labels and adjust positioning
    geom_text(aes(label = sprintf("%.0f", cfr * 100)), vjust = -0.5, size = 2) +
  
#Set theme
  theme_bw() +
  
#Adjust the positioning of the labels and remove panel grids
  theme(axis.text.x = element_text(angle = 90, hjust = 1), plot.title = element_text(hjust = 0), panel.grid.major = element_blank(),panel.grid.minor = element_blank(), legend.position = "none")

```

```{r echo=FALSE, out.width="100%"}
knitr::include_graphics(here::here("images", "cfr-rural-urban.png"), error = FALSE)
```

```{r}

#Create a new column to identify category names according to urban/rural
urban_rural <- combined_data %>%
  mutate(category = ifelse(x2013_code %in% c(1, 2, 3, 4), "Urban", "Rural"))

#Filter for urban values
urban_data <- urban_rural %>% 
  filter(category == "Urban")

#Filter for rural values
rural_data <- urban_rural %>% 
  filter(category == "Rural")

# Calculate CFR % for urban and rural areas
urban_cfr <- urban_data %>%
  
#Group by case month
  group_by(case_month) %>%
  
#Compute for the CFR %
  summarize(cfr = sum(count.x) / sum(count.y))

#Calculate the CFR % for rural areas and store into a variable
rural_cfr <- rural_data %>%
  
#Group by case month
  group_by(case_month) %>%
  
#Compute for the CFR %
  summarize(cfr = sum(count.x) / sum(count.y))

#Combine the CFR % and for both urban and rural areas 
cfr_data <- bind_rows(
  urban_cfr %>% mutate(category = "Urban"),
  rural_cfr %>% mutate(category = "Rural")
)

#Use ggplot to create the graph
cfr_data %>% 
  ggplot() +
  
#Set case month as the x-axis, CFR % as the y-axis, and the category name as the grouping and color
  aes(x = case_month, y = cfr, color = category, group = category) +
  
#Create the line graph
  geom_line() +
  
#Include data labels and adjusting positioning
  geom_text(aes(label = sprintf("%.1f", cfr * 100)), vjust = -0.5, size = 2) +
  
#Include labels for the graph
  labs(x = "", y = "", color = "Category", title = "Covid CFR % by rural and urban areas") +
  
#Set theme
  theme_bw() +
  
#Adjust the positioning of the labels and remove panel grid
  theme(axis.text.x = element_text(angle = 90, hjust = 1), plot.title = element_text(hjust = 0), panel.grid.major = element_blank(),panel.grid.minor = element_blank())

```

# Money in US politics

```{r}
#| label: allow-scraping-opensecrets
#| warning: false
#| message: false

library(robotstxt)
paths_allowed("https://www.opensecrets.org")

base_url <- "https://www.opensecrets.org/political-action-committees-pacs/foreign-connected-pacs/2022"

contributions_tables <- base_url %>%
  read_html() 

```

-   First, make sure you can scrape the data for 2022. Use janitor::clean_names() to rename variables scraped using `snake_case` naming.

```{r}

#Rename variables using snake_case naming
contributions_tables <- contributions_tables %>% 
  janitor::clean_names(case = "snake")

```

-   Clean the data:

```{r}
# write a function to parse_currency
parse_currency <- function(x){
  x %>%
    
    # remove dollar signs
    str_remove("\\$") %>%
    
    # remove all occurrences of commas
    str_remove_all(",") %>%
    
    # convert to numeric
    as.numeric()
}

#Extract the contributions table from the web page
contributions <- contributions_tables %>% 
  
#Select table element
  html_element("#main > div.Main-wrap.l-padding.u-mt2 > div > div > div.l-primary > div:nth-child(1) > div > div:nth-child(5)") %>% 
  
#Convert to dataframe
  html_table()

#Use janitor to clean column names
contributions <- contributions %>% 
  janitor::clean_names()

# clean country/parent co and contributions 
contributions <- contributions %>%
  separate(country_of_origin_parent_company, 
           into = c("country", "parent"), 
           sep = "/", 
           extra = "merge") %>%
  mutate(
    total = parse_currency(total),
    dems = parse_currency(dems),
    repubs = parse_currency(repubs)
  )
```

-   Write a function called `scrape_pac()` that scrapes information from the Open Secrets webpage for foreign-connected PAC contributions in a given year. 
    
```{r}

#Create a function to scrape information from the Open Secrets webpage
scrape_pac <- function(url) {
  #Extract year from URL
  year <- str_sub(url, -4)
  
  #Print the URL being processed
  cat("Scraping data for year:", year, "\n")
  
  #Read the HTML content
  page <- read_html(url)
  
  #Extract the table
  table <- page %>% 
    html_table(header = TRUE) %>% 
    .[[1]]
  
  #Clean column names
  table <- table %>% 
    janitor::clean_names()
  
  #Convert contribution amounts to numeric
  table <- table %>% 
    mutate(
      total = parse_currency(total),
      dems = parse_currency(dems),
      repubs = parse_currency(repubs)
    )
  
  #Add year column
  table$year <- year
  
  return(table)
}

```

-   Define the URLs for 2022, 2020, and 2000 contributions. Then, test your function using these URLs as inputs. Does the function seem to do what you expected it to do?

```{r}

#Define the URLs
url_2022 <- "https://www.opensecrets.org/political-action-committees-pacs/foreign-connected-pacs/2022"
url_2020 <- "https://www.opensecrets.org/political-action-committees-pacs/foreign-connected-pacs/2020"
url_2000 <- "https://www.opensecrets.org/political-action-committees-pacs/foreign-connected-pacs/2000"

#Test the function for 2022 contributions
data_2022 <- scrape_pac(url_2022)
print(head(data_2022))

#Test the function for 2020 contributions
data_2020 <- scrape_pac(url_2020)
print(head(data_2020))

#Test the function for 2000 contributions
data_2000 <- scrape_pac(url_2000)
print(head(data_2000))

```

-   Construct a vector called `urls` that contains the URLs for each webpage that contains information on foreign-connected PAC contributions for a given year.

```{r}

#Define the base URL
base_url <- "https://www.opensecrets.org/political-action-committees-pacs/foreign-connected-pacs/"

#Define the years
years <- c(2022, 2020, 2000)

#Construct the vector of URLs
urls <- paste0(base_url, years)

```

-   Map the `scrape_pac()` function over `urls` in a way that will result in a data frame called `contributions_all`.

```{r}

#Map the scrape_pac function over the URLs
contributions_all <- map_dfr(urls, scrape_pac)

```

-   Write the data frame to a csv file called `contributions-all.csv` in the `data` folder.

```{r}

#Set the file path for the CSV file
file_path <- "data/contributions-all.csv"

#Write the data frame to a CSV file
write.csv(contributions_all, file = file_path, row.names = FALSE)

```

# Scraping consulting jobs

The website [https://www.consultancy.uk/jobs/](https://www.consultancy.uk/jobs) lists job openings for consulting jobs.

```{r}
#| label: consulting_jobs_url
#| eval: false

library(robotstxt)
paths_allowed("https://www.consultancy.uk") #is it ok to scrape?

base_url <- "https://www.consultancy.uk/jobs/page/1"

listings_html <- base_url %>%
  read_html()

```

-   Write a function called `scrape_jobs()` that scrapes information from the webpage for consulting positions.

```{r}

#Create a function that would scrape information on the Consultancy UK webpage
scrape_jobs <- function(url) {
  
  #Read the HTML content of the webpage
  listings_html <- read_html(url)
  
  #Extract information from the HTML content using CSS selectors and clean the texts if necessary
  job <- listings_html %>% 
    html_nodes(css = ".title") %>% 
    html_text()
  
  firm <- listings_html %>% 
    html_nodes(css = "td.hide-phone") %>% 
    html_text() %>% 
    str_remove("\n")
  
  functional_area <- listings_html %>% 
    html_nodes(css = "td.hide-tablet-and-less") %>%
    html_text() %>%
    str_remove_all("\n|\\+1|\\+2|\\+3|\\+4|\\+5|\\+6") %>%
    str_replace_all("(?<=[a-z])(?=[A-Z])", " ")

  type <- listings_html %>% 
    html_nodes(css = "td.hide-tablet-landscape") %>%
    html_text() %>% 
    str_trim()
  
  # Create a data frame with the extracted information
  job_data <- data.frame(job, firm, functional_area, type)
  
  return(job_data)
}

#Test the scrape_jobs() function with a specific URL
url <- "https://www.consultancy.uk/jobs/page/1"
job_data <- scrape_jobs(url)

#Print the scraped job data
print(job_data)

#Test the function with other pages, e.g., https://www.consultancy.uk/jobs/page/2
url_2 <- "https://www.consultancy.uk/jobs/page/2"
job_data_2 <- scrape_jobs(url_2)

#Print the scraped job data from page 2
print(job_data_2)

#Join strings using str_c()
page <- 8
url <- str_c("https://www.consultancy.uk/jobs/page/", page)

```

```         
base_url <- "https://www.consultancy.uk/jobs/page/1"
url <- str_c(base_url, page)
```

-   Construct a vector called `pages` that contains the numbers for each page available

```{r}

#Create a vector of pages
pages <- seq(1,8)

```

-   Map the `scrape_jobs()` function over `pages` in a way that will result in a data frame called `all_consulting_jobs`.

```{r}

# Define the URL pattern
base_url <- "https://www.consultancy.uk/jobs/page/"

# Map the scrape_jobs() function over pages and combine the results into a data frame
all_consulting_jobs <- map_df(pages, ~scrape_jobs(str_c(base_url, .x)))

# Print the resulting data frame
print(all_consulting_jobs)

```

-   Write the data frame to a csv file called `all_consulting_jobs.csv` in the `data` folder.

```{r}

#Define the file path
file_path <- "data/all_consulting_jobs.csv"

#Write the data frame to the CSV file
write_csv(all_consulting_jobs, file_path)

```

# Details

-   Who did you collaborate with: N/A
-   Approximately how much time did you spend on this problem set: three days
=======
---
title: 'Homework 3: Databases, web scraping, and a basic Shiny app'
author: "MIKE DIZON"
date: "`r Sys.Date()`"
output:
  word_document:
    toc: yes
  pdf_document:
    toc: yes
  html_document:
    theme: flatly
    highlight: zenburn
    number_sections: yes
    toc: yes
    toc_float: yes
    code_folding: show
---

```{r}
#| label: load-libraries
#| echo: false # This option disables the printing of code (only output is displayed).
#| message: false
#| warning: false

library(tidyverse)
library(wbstats)
library(tictoc)
library(skimr)
library(countrycode)
library(here)
library(DBI)
library(dbplyr)
library(arrow)
library(rvest)
library(robotstxt) # check if we're allowed to scrape the data
library(scales)
library(sf)
library(readxl)
library(stringr)
```

# Money in UK politics

[The Westminster Accounts](https://news.sky.com/story/the-westminster-accounts-12786091), a recent collaboration between Sky News and Tortoise Media, examines the flow of money through UK politics. It does so by combining data from three key sources:

1.  [Register of Members' Financial Interests](https://www.parliament.uk/mps-lords-and-offices/standards-and-financial-interests/parliamentary-commissioner-for-standards/registers-of-interests/register-of-members-financial-interests/),
2.  [Electoral Commission records of donations to parties](http://search.electoralcommission.org.uk/English/Search/Donations), and
3.  [Register of All-Party Parliamentary Groups](https://www.parliament.uk/mps-lords-and-offices/standards-and-financial-interests/parliamentary-commissioner-for-standards/registers-of-interests/register-of-all-party-party-parliamentary-groups/).

You can [search and explore the results](https://news.sky.com/story/westminster-accounts-search-for-your-mp-or-enter-your-full-postcode-12771627) through the collaboration's interactive database. Simon Willison [has extracted a database](https://til.simonwillison.net/shot-scraper/scraping-flourish) and this is what we will be working with. If you want to read more about [the project's methodology](https://www.tortoisemedia.com/2023/01/08/the-westminster-accounts-methodology/).

## Open a connection to the database

The database made available by Simon Willison is an `SQLite` database

```{r}
sky_westminster <- DBI::dbConnect(
  drv = RSQLite::SQLite(),
  dbname = here::here("data", "sky-westminster-files.db")
)
```

How many tables does the database have?

```{r}

#Identify the number of tables in the remote database
num_tables <- length(DBI::dbListTables(sky_westminster))

```

**Answer:** The are 7 tables in the sky_westminster database.

## Which MP has received the most amount of money?

```{r}

#Browse tables in the database
DBI::dbListTables(sky_westminster)

#Store the 'payments' table as a database object
payments_db <- dplyr::tbl(sky_westminster, "payments")

#Glimpse payments_db to check for contents
glimpse(payments_db)

#Store the 'members' table as a database object
members_db <- dplyr::tbl(sky_westminster, "members")

#Glimpse members_db to check for contents
glimpse(members_db)

#Compute for the amounts that each MP received
mp_amount_summary <- payments_db %>% 
  
#Group by member id
  group_by(member_id) %>%
  
#Compute for the sum of donations for each member id
  summarize(total_value = sum(value)) %>% 
  
#Arrange in descending order
  arrange(desc(total_value))

#Extract the member ID of the MP with the most amount
mp_with_most_money <- mp_amount_summary %>% 
  head(1)

#Identify the name of the MP using left_join
mp_name <- left_join(mp_with_most_money, members_db, by = c("member_id" = "id")) %>% 
  collect()

```

**Answer:** Theresa May was the MP who has received the most amount of money with 2809765.

## Any `entity` that accounts for more than 5% of all donations?

Is there any `entity` whose donations account for more than 5% of the total payments given to MPs over the 2020-2022 interval? Who are they and who did they give money to?

```{r}

#Identify sum of total payments made to MPs and store into a variable
total_payments <- payments_db %>% 
  summarize(total_payments = sum(value)) %>% 
  
#Extract only the value for total payments
  pull(total_payments)

#Compute for donations made by each entity and store into a variable
entity_donations <- payments_db %>% 
  
#Group by entity name and member id of the MP
  group_by(entity, member_id) %>% 

#Compute for the total value of payments for each entity
  summarize(total_donations = sum(value)) %>% 
  
#Add a new column for the percentage of the donation for each entity over the total payments
  mutate(percentage = (total_donations / total_payments) * 100, .groups = 'drop') %>% 
  
#Ungroup data
  ungroup() %>%
  
#Use left join to combine with the members_db dataset
  left_join(collect(members_db), by = c("member_id" = "id"), copy = TRUE)

#Identify entities that have made the highest contributions and store into a variable
high_donation_entities <- entity_donations %>%
  
#Filter for the entities that made more than 5% in contributions
  filter(percentage > 5) %>% 
  collect()

```

**Answer:** Withers LLP made more than 5% in donations or 1812732 in nominal amount to Sir Geoffrey Cox over the 2020-2022 period.

## Do `entity` donors give to a single party or not?

-   How many distinct entities who paid money to MPs are there?
-   How many (as a number and %) donated to MPs belonging to a single party only?

```{r}

#Store the 'parties' table as a database object
parties_db <- dplyr::tbl(sky_westminster, "parties")

#Store the 'party_donations' table as a database object
party_donations_db <- dplyr::tbl(sky_westminster, "party_donations")

#Determine the number of distinct entities and store into a variable
num_entities <- party_donations_db %>% 
  
#Identify the distinct entities
  distinct(entity) %>% 
  
#Count the number of distinct entities
  summarize(num_entities = n()) %>% 
  
#Extract the value of the number of entities
  pull(num_entities)


#Identify the entities that donated to a single party and store into a variable
entities_single_party <- party_donations_db %>% 
  
#Group by entity
  group_by(entity) %>%
  
#Compute for the number of distinct party id
  summarize(num_parties = n_distinct(party_id)) %>% 
  
#Filter to obtain the entity that donated to a single party
  filter(num_parties == 1)

#Identify the number of entities that donated to a single party and store into a variable
num_entites_single_party <- entities_single_party %>%
  
#Compute for the number of these entities
  summarize(num_entites_single_party = n()) %>% 
  
#Extract only the number of these entities
  pull(num_entites_single_party)

#Compute for the percentage and store into a variable
percentage_entities_single_party <- (num_entites_single_party/num_entities)*100

```

**Answer:** There are 1077 distinct entities that donated money to MPs. Out of these entities, 1068 (99%) donated to MPs belonging to a single party.

## Which party has raised the greatest amount of money in each of the years 2020-2022?

```{r}

#Compute for the total amounts raised per year by each party and store into a variable
party_year_amounts <- party_donations_db %>%
  
#Group by year and party id
  group_by(year = substr(date, 1, 4), party_id) %>%
  
#Compute for the sum for each party per year
  summarise(total_amount = sum(value), .groups = "drop")

#Identify the party with the highest amount per year and store into a variable
max_party_per_year <- party_year_amounts %>%
  
#Group by year
  group_by(year) %>%
  
#Filter for the maximum amount
  filter(total_amount == max(total_amount)) %>%
  
#Use inner join to combine with the parties_db dataset
  inner_join(parties_db, by = c("party_id" = "id"), copy = TRUE) %>% 
  
#Select relevant columns
  select(year, party_name = name, total_amount) %>%
  
#Collect the data
  collect()

```

**Answer:** For each of the years from 2020 to 2022, the Conservative party has raised the most amount of money: 42770782 for 2020, 17718212 for 2021, and 15568476 for 2022.

I would like you to write code that generates the following table.

```{r echo=FALSE, out.width="80%"}
knitr::include_graphics(here::here("images", "total_donations_table.png"), error = FALSE)
```

```{r}

#Use inner join to combine the party_donations_db and parties_db datasets
result <- party_donations_db %>%
  inner_join(parties_db, by = c("party_id" = "id")) %>%
  
#Group by year and party name
  group_by(year = substr(date, 1, 4), name) %>%
  
#Compute for the total donations per year for each party
  summarise(total_year_donations = sum(value)) 

#Calculate the total year donations for each year and store into a variable
total_year_donations <- result %>%
  
#Group by year
  group_by(year) %>%
  
#Compute for the total donations per year and convert to numeric
  summarise(total_donations = sum(as.numeric(total_year_donations)))

#Use inner join to combine data with total yearly donations by year
result <- result %>%
  inner_join(total_year_donations, by = "year", copy = TRUE) %>%
  
#Add another column for the proportion of total donations per party by the total donations per year
  mutate(
    prop = as.numeric(total_year_donations) / total_donations
  ) %>%
  
#Ungroup the data
  ungroup() %>%
  
#Remove the total donations column
  select(-total_donations) %>% 
  
#Collect the data
  collect()

```

... and then, based on this data, plot the following graph.

```{r echo=FALSE, out.width="80%"}
knitr::include_graphics(here::here("images", "total_donations_graph.png"), error = FALSE)
```

```{r}

#Arrange amount of total donations per year by party in descending order
result <- result %>%
  mutate(name = fct_reorder(name, total_year_donations, .desc = TRUE))

#Use ggplot to create graph
result %>% 
ggplot() +
  
#Set year as the x-axis, total donations per year by party as the y-axis, and the party name as the fill
  aes(x = year, y = total_year_donations, fill = name) +
  
#Create the bar graph and set position
  geom_col(position = "dodge") +
  
#Create labels for the graph
  labs(x = "", y = "", 
       title = "Conservatives have captured the majority of political donations",
       subtitle = "Donations to political parties, 2020-2022", fill = "Party") +
  
#Set theme
  theme_minimal() +
  
#Make the labels of the y-axis into integers
  scale_y_continuous(labels = scales::comma)

```

```{r}
dbDisconnect(sky_westminster)
```

# Anonymised Covid patient data from the CDC

```{r}
#| echo: false
#| message: false
#| warning: false


tic() # start timer
cdc_data <- open_dataset(here::here("data", "cdc-covid-geography"))
toc() # stop timer


glimpse(cdc_data)
```

Can you query the database and replicate the following plot?

```{r echo=FALSE, out.width="100%"}
knitr::include_graphics(here::here("images", "covid-CFR-ICU.png"), error = FALSE)
```

```{r}

#Calculate the number of deaths from the database and store into a variable
deaths_data <- cdc_data %>%
  
#Filter the data to select relevant columns within age group, sex, and ICU admission
  filter(sex %in% c("Male", "Female"), icu_yn %in% c("Yes", "No"), death_yn == "Yes") %>%
  select(age_group, sex, icu_yn)

#Group the number of deaths and store into a variable
grouped_deaths <- deaths_data %>%
  
#Group by sex, age group, and ICU admission
  group_by(sex, age_group, icu_yn) %>%
  
#Count the number of rows
  summarize(count = n()) %>%
  
#Collect the data
  collect()

#Calculate the number of cases from the database and store into a variable
cases_data <- cdc_data %>%
  
#Filter the data to select relevant columns within age group, sex, and ICU admission
  filter(sex %in% c("Male", "Female"), icu_yn %in% c("Yes", "No")) %>%
  select(age_group, sex, icu_yn)

#Group the number of deaths and store into a variable
grouped_cases <- cases_data %>%
  
#Group by sex, age group, and ICU admission
  group_by(sex, age_group, icu_yn) %>%
  
#Count the number of rows
  summarize(count = n()) %>%
  
#Collect the data
  collect()

#Calculate the CFR % and store into a variable
CFR_data <- grouped_deaths %>% 
  
#Use left join the combine with the grouped cases
  left_join(grouped_cases, by = c("sex", "age_group", "icu_yn")) %>% 
  
#Add another column to compute for the CFR %
  mutate(cfr = (count.x / count.y))

#Rename the data to ICU Admission/No ICU Admission
CFR_data <- CFR_data %>%
  mutate(icu_yn = recode(icu_yn, "Yes" = "ICU Admission", "No" = "No ICU Admission"))

#Plot the CFR % data using ggplot
CFR_data %>% 
ggplot() +
  
#Use the CFR % as the x-axis and age group as the y-axis
  aes(x = cfr, y = age_group) +
  
#Create the bar graph
  geom_bar(stat = "identity", fill = "#FF8F7C") +
  
#Include data labels in the graph and set desired positioning
  geom_text(aes(label = sprintf("%.0f", cfr * 100)), vjust = 0.5, hjust = 1, size = 2) +
  
#Facet grid by ICU Admission and sex
  facet_grid(icu_yn ~ sex, scales = "free", space = "free") +
  
#Remove axis labels and include title for the graph
  labs(x = "", y = "", title = "Covid CFR % by age group, sex and ICU Admission") +
  
#Set theme
  theme_bw() + 
  
#Set scale for the x-axis and title positioning
  scale_x_continuous(labels = scales::percent) +
  theme(plot.title = element_text(hjust = 0))


```

The previous plot is an aggregate plot for all three years of data. What if we wanted to plot Case Fatality Ratio (CFR) over time? Write code that collects the relevant data from the database and plots the following

```{r echo=FALSE, out.width="100%"}
knitr::include_graphics(here::here("images", "cfr-icu-overtime.png"), error = FALSE)
```

```{r}

#Calculate the number of deaths from the database and store into a variable
deaths_data1 <- cdc_data %>%
  
#Filter the data and select relevant columns within case month, age group, sex, and ICU admission
  filter(sex %in% c("Male", "Female"), icu_yn %in% c("Yes", "No"), death_yn == "Yes") %>%
  select(case_month, age_group, sex, icu_yn)

#Group the number of deaths and store into a variable
grouped_deaths1 <- deaths_data1 %>%
  
#Group by case month, sex, age group, and ICU admission
  group_by(case_month, sex, age_group, icu_yn) %>%
  
#Count the number of rows
  summarize(count = n()) %>%
  
#Collect the data
  collect()

#Calculate the number of cases and store into a variable
cases_data1 <- cdc_data %>%
  
#Filter the data and select relevant columns within case month, age group, sex, and ICU admission
  filter(sex %in% c("Male", "Female"), icu_yn %in% c("Yes", "No")) %>%
  select(case_month, age_group, sex, icu_yn)

#Group the number of cases and store into a variable
grouped_cases1 <- cases_data1 %>%
  
#Group by case month, sex, age group, and ICU admission
  group_by(case_month, sex, age_group, icu_yn) %>%
  
#Count the number of rows
  summarize(count = n()) %>%
  
#Collect the data
  collect()

#Calculate the CFR% and store into a variable
CFR_data1 <- grouped_deaths1 %>% 
  
#Use left join to combine with grouped cases
  left_join(grouped_cases1, by = c("case_month", "sex", "age_group", "icu_yn")) %>% 
  
#Add another column to compute for the CFR %
  mutate(cfr = (count.x / count.y))

#Rename the data to ICU Admission/No ICU Admission
CFR_data1 <- CFR_data1 %>%
  mutate(icu_yn = recode(icu_yn, "Yes" = "ICU Admission", "No" = "No ICU Admission"))

#Plot the CFR % data using ggplot
CFR_data1 %>% 
ggplot() +

#Set case month as the x-axis, CFR % as the y-axis, and age group as the grouping category
  aes(x = case_month, y = cfr, group = age_group, color = age_group) +
  
#Create the line graph
  geom_line() +
  
#Include data labels and set positioning
  geom_text(aes(label = sprintf("%.0f", cfr * 100)), vjust = -0.5, size = 3) +
  
#Use facet grid with ICU Admission and sex
  facet_grid(icu_yn ~ sex, scales = "free_y", space = "free") +
  
#Remove axis labels and include title for the graph
  labs(x = "", y = "", color = "Age Group", title = "Covid CFR % by age group, sex and ICU Admission") +
  
#Set theme
  theme_bw() +
  
#Adjust positioning of the case month and plot title, and remove the panel grids
  theme(axis.text.x = element_text(angle = 90, hjust = 1), plot.title = element_text(hjust = 0), panel.grid.major = element_blank(),panel.grid.minor = element_blank())

```

```{r}
urban_rural <- read_xlsx(here::here("data", "NCHSURCodes2013.xlsx")) %>% 
  janitor::clean_names() 
```

Can you query the database, extract the relevant information, and reproduce the following two graphs that look at the Case Fatality ratio (CFR) in different counties, according to their population?

```{r echo=FALSE, out.width="100%"}
knitr::include_graphics(here::here("images", "cfr-county-population.png"), error = FALSE)
```

```{r}

#Calculate the number of deaths from the database and store into a variable
deaths_data2 <- cdc_data %>%
  
#Filter for deaths
  filter(death_yn == "Yes") %>%
  
#Select relevant columns
  select(case_month, county_fips_code)

#Group the deaths and store into a variable
grouped_deaths2 <- deaths_data2 %>%
  
#Group by case month and county fips code
  group_by(case_month, county_fips_code) %>%
  
#Count the number of deaths per case month and county fips code
  summarize(count = n()) %>%
  
#Collect the data
  collect()

#Calculate the total case from the database and store into a variable
cases_data2 <- cdc_data %>%
  
#Select the relevant columns
  select(case_month, county_fips_code)

#Group the cases and store into a variable
grouped_cases2 <- cases_data2 %>%
  
#Group by case month and country fips code
  group_by(case_month, county_fips_code) %>%
  
#Count the number of cases by case month and county fips code
  summarize(count = n()) %>%
  
#Collect the data
  collect()

#Combine grouped deaths and cases into a single dataframe
CFR_data2 <- grouped_deaths2 %>% 
  
#Use left join to combined the data by case month and county fips code
  left_join(grouped_cases2, by = c("case_month", "county_fips_code")) %>% 
  
#Add new column to compute for the CFR %
  mutate(cfr = (count.x / count.y))

#Combine CFR data with the urban_rural dataframe
combined_data <- left_join(CFR_data2, urban_rural, by = c("county_fips_code" = "fips_code")) %>% 
  
#Filter out the NA values
  filter(!is.na(x2013_code)) 

#Group the combined data and store into a new variable
combined_data1 <- combined_data %>% 
  
#Group by case month, country fips code, and category name
  group_by(case_month, county_fips_code, x2013_code)%>% 
  
#Compute the CFR %
  summarize(cfr = sum(count.x) / sum(count.y))

#Use ggplot to graph the data
combined_data1 %>% 
  ggplot() +
  
#Set case month as the x-axis, CFR % as the y-axis, and the category name as the grouping as color
  aes(x = case_month, y = cfr, group = x2013_code, color = x2013_code) +
  
#Create the line graph
  geom_line() +
  
#Use facet wrap to create grids by category name and recode the category names
  facet_wrap(~ x2013_code, ncol = 2, scales = "free_y", labeller = labeller(x2013_code = c("1" = "1.Large central metro", "2" = "2.Large fringe metro", "3" = "3.Medium metro", "4" = "4.Small metropolitan", "5" = "5.Micropolitan", "6" = "6.Noncore"))) +
  
#Include labels for the graph
  labs(x = "", y = "", color = "x2013_code", title = "Covid CFR % by county population") +
  
#Include data labels and adjust positioning
    geom_text(aes(label = sprintf("%.0f", cfr * 100)), vjust = -0.5, size = 2) +
  
#Set theme
  theme_bw() +
  
#Adjust the positioning of the labels and remove panel grids
  theme(axis.text.x = element_text(angle = 90, hjust = 1), plot.title = element_text(hjust = 0), panel.grid.major = element_blank(),panel.grid.minor = element_blank(), legend.position = "none")

```

```{r echo=FALSE, out.width="100%"}
knitr::include_graphics(here::here("images", "cfr-rural-urban.png"), error = FALSE)
```

```{r}

#Create a new column to identify category names according to urban/rural
urban_rural <- combined_data %>%
  mutate(category = ifelse(x2013_code %in% c(1, 2, 3, 4), "Urban", "Rural"))

#Filter for urban values
urban_data <- urban_rural %>% 
  filter(category == "Urban")

#Filter for rural values
rural_data <- urban_rural %>% 
  filter(category == "Rural")

# Calculate CFR % for urban and rural areas
urban_cfr <- urban_data %>%
  
#Group by case month
  group_by(case_month) %>%
  
#Compute for the CFR %
  summarize(cfr = sum(count.x) / sum(count.y))

#Calculate the CFR % for rural areas and store into a variable
rural_cfr <- rural_data %>%
  
#Group by case month
  group_by(case_month) %>%
  
#Compute for the CFR %
  summarize(cfr = sum(count.x) / sum(count.y))

#Combine the CFR % and for both urban and rural areas 
cfr_data <- bind_rows(
  urban_cfr %>% mutate(category = "Urban"),
  rural_cfr %>% mutate(category = "Rural")
)

#Use ggplot to create the graph
cfr_data %>% 
  ggplot() +
  
#Set case month as the x-axis, CFR % as the y-axis, and the category name as the grouping and color
  aes(x = case_month, y = cfr, color = category, group = category) +
  
#Create the line graph
  geom_line() +
  
#Include data labels and adjusting positioning
  geom_text(aes(label = sprintf("%.1f", cfr * 100)), vjust = -0.5, size = 2) +
  
#Include labels for the graph
  labs(x = "", y = "", color = "Category", title = "Covid CFR % by rural and urban areas") +
  
#Set theme
  theme_bw() +
  
#Adjust the positioning of the labels and remove panel grid
  theme(axis.text.x = element_text(angle = 90, hjust = 1), plot.title = element_text(hjust = 0), panel.grid.major = element_blank(),panel.grid.minor = element_blank())

```

# Money in US politics

```{r}
#| label: allow-scraping-opensecrets
#| warning: false
#| message: false

library(robotstxt)
paths_allowed("https://www.opensecrets.org")

base_url <- "https://www.opensecrets.org/political-action-committees-pacs/foreign-connected-pacs/2022"

contributions_tables <- base_url %>%
  read_html() 

```

-   First, make sure you can scrape the data for 2022. Use janitor::clean_names() to rename variables scraped using `snake_case` naming.

```{r}

#Rename variables using snake_case naming
contributions_tables <- contributions_tables %>% 
  janitor::clean_names(case = "snake")

```

-   Clean the data:

```{r}
# write a function to parse_currency
parse_currency <- function(x){
  x %>%
    
    # remove dollar signs
    str_remove("\\$") %>%
    
    # remove all occurrences of commas
    str_remove_all(",") %>%
    
    # convert to numeric
    as.numeric()
}

#Extract the contributions table from the web page
contributions <- contributions_tables %>% 
  
#Select table element
  html_element("#main > div.Main-wrap.l-padding.u-mt2 > div > div > div.l-primary > div:nth-child(1) > div > div:nth-child(5)") %>% 
  
#Convert to dataframe
  html_table()

#Use janitor to clean column names
contributions <- contributions %>% 
  janitor::clean_names()

# clean country/parent co and contributions 
contributions <- contributions %>%
  separate(country_of_origin_parent_company, 
           into = c("country", "parent"), 
           sep = "/", 
           extra = "merge") %>%
  mutate(
    total = parse_currency(total),
    dems = parse_currency(dems),
    repubs = parse_currency(repubs)
  )
```

-   Write a function called `scrape_pac()` that scrapes information from the Open Secrets webpage for foreign-connected PAC contributions in a given year. 
    
```{r}

#Create a function to scrape information from the Open Secrets webpage
scrape_pac <- function(url) {
  #Extract year from URL
  year <- str_sub(url, -4)
  
  #Print the URL being processed
  cat("Scraping data for year:", year, "\n")
  
  #Read the HTML content
  page <- read_html(url)
  
  #Extract the table
  table <- page %>% 
    html_table(header = TRUE) %>% 
    .[[1]]
  
  #Clean column names
  table <- table %>% 
    janitor::clean_names()
  
  #Convert contribution amounts to numeric
  table <- table %>% 
    mutate(
      total = parse_currency(total),
      dems = parse_currency(dems),
      repubs = parse_currency(repubs)
    )
  
  #Add year column
  table$year <- year
  
  return(table)
}

```

-   Define the URLs for 2022, 2020, and 2000 contributions. Then, test your function using these URLs as inputs. Does the function seem to do what you expected it to do?

```{r}

#Define the URLs
url_2022 <- "https://www.opensecrets.org/political-action-committees-pacs/foreign-connected-pacs/2022"
url_2020 <- "https://www.opensecrets.org/political-action-committees-pacs/foreign-connected-pacs/2020"
url_2000 <- "https://www.opensecrets.org/political-action-committees-pacs/foreign-connected-pacs/2000"

#Test the function for 2022 contributions
data_2022 <- scrape_pac(url_2022)
print(head(data_2022))

#Test the function for 2020 contributions
data_2020 <- scrape_pac(url_2020)
print(head(data_2020))

#Test the function for 2000 contributions
data_2000 <- scrape_pac(url_2000)
print(head(data_2000))

```

-   Construct a vector called `urls` that contains the URLs for each webpage that contains information on foreign-connected PAC contributions for a given year.

```{r}

#Define the base URL
base_url <- "https://www.opensecrets.org/political-action-committees-pacs/foreign-connected-pacs/"

#Define the years
years <- c(2022, 2020, 2000)

#Construct the vector of URLs
urls <- paste0(base_url, years)

```

-   Map the `scrape_pac()` function over `urls` in a way that will result in a data frame called `contributions_all`.

```{r}

#Map the scrape_pac function over the URLs
contributions_all <- map_dfr(urls, scrape_pac)

```

-   Write the data frame to a csv file called `contributions-all.csv` in the `data` folder.

```{r}

#Set the file path for the CSV file
file_path <- "data/contributions-all.csv"

#Write the data frame to a CSV file
write.csv(contributions_all, file = file_path, row.names = FALSE)

```

# Scraping consulting jobs

The website [https://www.consultancy.uk/jobs/](https://www.consultancy.uk/jobs) lists job openings for consulting jobs.

```{r}
#| label: consulting_jobs_url
#| eval: false

library(robotstxt)
paths_allowed("https://www.consultancy.uk") #is it ok to scrape?

base_url <- "https://www.consultancy.uk/jobs/page/1"

listings_html <- base_url %>%
  read_html()

```

-   Write a function called `scrape_jobs()` that scrapes information from the webpage for consulting positions.

```{r}

#Create a function that would scrape information on the Consultancy UK webpage
scrape_jobs <- function(url) {
  
  #Read the HTML content of the webpage
  listings_html <- read_html(url)
  
  #Extract information from the HTML content using CSS selectors and clean the texts if necessary
  job <- listings_html %>% 
    html_nodes(css = ".title") %>% 
    html_text()
  
  firm <- listings_html %>% 
    html_nodes(css = "td.hide-phone") %>% 
    html_text() %>% 
    str_remove("\n")
  
  functional_area <- listings_html %>% 
    html_nodes(css = "td.hide-tablet-and-less") %>%
    html_text() %>%
    str_remove_all("\n|\\+1|\\+2|\\+3|\\+4|\\+5|\\+6") %>%
    str_replace_all("(?<=[a-z])(?=[A-Z])", " ")

  type <- listings_html %>% 
    html_nodes(css = "td.hide-tablet-landscape") %>%
    html_text() %>% 
    str_trim()
  
  # Create a data frame with the extracted information
  job_data <- data.frame(job, firm, functional_area, type)
  
  return(job_data)
}

#Test the scrape_jobs() function with a specific URL
url <- "https://www.consultancy.uk/jobs/page/1"
job_data <- scrape_jobs(url)

#Print the scraped job data
print(job_data)

#Test the function with other pages, e.g., https://www.consultancy.uk/jobs/page/2
url_2 <- "https://www.consultancy.uk/jobs/page/2"
job_data_2 <- scrape_jobs(url_2)

#Print the scraped job data from page 2
print(job_data_2)

#Join strings using str_c()
page <- 8
url <- str_c("https://www.consultancy.uk/jobs/page/", page)

```

```         
base_url <- "https://www.consultancy.uk/jobs/page/1"
url <- str_c(base_url, page)
```

-   Construct a vector called `pages` that contains the numbers for each page available

```{r}

#Create a vector of pages
pages <- seq(1,8)

```

-   Map the `scrape_jobs()` function over `pages` in a way that will result in a data frame called `all_consulting_jobs`.

```{r}

# Define the URL pattern
base_url <- "https://www.consultancy.uk/jobs/page/"

# Map the scrape_jobs() function over pages and combine the results into a data frame
all_consulting_jobs <- map_df(pages, ~scrape_jobs(str_c(base_url, .x)))

# Print the resulting data frame
print(all_consulting_jobs)

```

-   Write the data frame to a csv file called `all_consulting_jobs.csv` in the `data` folder.

```{r}

#Define the file path
file_path <- "data/all_consulting_jobs.csv"

#Write the data frame to the CSV file
write_csv(all_consulting_jobs, file_path)

```
