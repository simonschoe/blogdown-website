---
title: "old post"
author: Simon Schölzel
date: 2020-06-11
output:
  html_document:
    keep_md: true
categories:
  - rmarkdown
  - rvest
  - stringdist
  - ggplot2
tags:
  - rmarkdown
  - rvest
  - stringdist
  - ggplot2
subtitle: ''
summary: "Data Privacy and Data Security -- two espoused concepts that gain substantial traction in the current times of Covid-19."
featured: no
image:
  caption: '[Photo by Scott Webb on Pexels](https://www.pexels.com/de-de/foto/ausrustung-burgersteig-gehweg-mauer-430208/)'
  focal_point: ''
  preview_only: no
projects: []
bibliography: references.bib
csl: "https://raw.githubusercontent.com/citation-style-language/styles/master/apa.csl"
---

Data Privacy and Data Security -- two espoused concepts that gain substantial traction in the current times of Covid-19, especially in the context of the upcoming German Corona tracking app [@SAP.2020] that is about to be released next week. Public expectations compete with individual reservations and data privacy concerns, partly fueled by events such as the 2013 Snowden revelations or the 2018 Camebridge Analytica scandal. Likewise, events like these contributed to the willingness of the EU to enforce supranational regulations on data privacy and security, climaxing the in the EU-GDPR [@EuropeanUnion.2018] which is binding for all EU member states since 2018-05-25.

In this blog post, I would like to take the current debate as an opportunity to dive into a dataset about monetary fines since the issuance of the EU-GDPR. In the process of doing so, I will do some web scraping to extract the dataset from the *GDPR Fines Tracker* on *privacyaffairs.com* [@PrivacyAffairs.2020], clean the dataset using some of the tools recently released as part of the `dplyr v1.0.0` release [@Wickham.2020], experiment with some fuzzy string matching capabilities provided in the `stringdist` package [@vanderLoo.2014] and engage in some visual EDA. Meanwhile, this post is inspired by [David Robinson](https://www.youtube.com/watch?v=EVvnnWKO_4w)'s TidyTuesday Series. The whole analysis is embeeded into an R markdown document.



### Load Packages


```r
if (!require("pacman")) install.packages("pacman")
pacman::p_load(rvest, tidyverse, kableExtra, extrafont, stringdist)
```

### Web Scraping

First, I extract the dataset from the *GDPR Fines Tracker* on *privacyaffairs.com* [@PrivacyAffairs.2020] using the `rvest` package and some regular expression (*regex*) to finally transform the JSON-format into a convenient data frame format.

```r
gdpr_data <- read_html("https://www.privacyaffairs.com/gdpr-fines/") %>% 
  #find html_node that contains the data
  html_nodes(xpath = "(.//script)[1]") %>% 
  #extract text
  rvest::html_text() %>%
  #trim to produce clean json format
  str_sub(start = str_locate(., "\\[")[[1]], end = str_locate(., "\\]")[[1]]) %>% 
  #remove new lines and carriage returns
  str_remove_all(paste("\\n", "\\r", sep = "|")) %>% 
  #parse JSON
  jsonlite::fromJSON()
```

In total, 273 violations are categorized by *PrivacyAffairs* as of the day of this blogpost (2020-06-11). I randomly sample five observations to get a first overview of the dataset.


```
>    id           name price
> 1   6        Romania 20000
> 2  86 United Kingdom     0
> 3 113          Spain   900
> 4  87 United Kingdom 80000
> 5 140        Hungary 15100
>                                                                                authority
> 1         Romanian National Supervisory Authority for Personal Data Processing (ANSPDCP)
> 2                                                               Information Commissioner
> 3                                               Spanish Data Protection Authority (AEPD)
> 4                                                               Information Commissioner
> 5 Hungarian National Authority for Data Protection and the Freedom of Information (NAIH)
>         date                                 controller
> 1 10/09/2019                           Vreau Credit SRL
> 2 07/10/2019 Driver and Vehicle Licensing Agency (DVLA)
> 3 11/07/2019                       TODOTECNICOS24H S.L.
> 4 07/16/2019                    Life at Parliament View
> 5 10/01/2019                            Town of Kerepes
>              articleViolated
> 1 Art. 32 GDPR, Art. 33 GDPR
> 2                    Unknown
> 3               Art. 13 GDPR
> 4   Data Protection Act 2018
> 5            Art. 6 (1) GDPR
>                                                                      type
> 1 Failure to implement sufficient measures to ensure information security
> 2                                            Non-compliance (Data Breach)
> 3                                   Information obligation non-compliance
> 4                                            Non-compliance (Data Breach)
> 5                    Non-compliance with lawful basis for data processing
>                                                                                                                                      source
> 1                                                                    https://www.dataprotection.ro/?page=Comunicat_Presa_09_10_2019&lang=ro
> 2              https://www.autoexpress.co.uk/car-news/consumer-news/91275/dvla-sale-of-driver-details-to-private-parking-firms-looked-at-by
> 3                                                                                    https://www.aepd.es/resoluciones/PS-00268-2019_ORI.pdf
> 4 https://ico.org.uk/about-the-ico/news-and-events/news-and-blogs/2019/07/estate-agency-fined-80-000-for-failing-to-keep-tenants-data-safe/
> 5                                                                                    https://www.naih.hu/files/NAIH-2019-2076-hatarozat.pdf
```

### Data Cleaning

In a next step, I streamline the dataset a little more and try to get rid of the various inconsistencies in the data entries. 

First, I would like to adjust some of the column names to highlight the actual content of the features. I rename `name` to `country`, `controller` to `entity` (i.e. the entity fined as a result of the violation) and `articleViolated` to `violation`. Second, I infer from the sample above that the violation `date` is not in standard international date format (`jjjj-mm-dd`). Let's change this using the `lubridate` package while properly accounting for 15 `NA`s (indicated by unix time `1970-01-01`). Third, I format the `price` feature by specyfing it as a proper currency (using the `scales` package). Fourth, and analogue to the violation `date`, I properly account for `NA`s -- taking the form "Not disclosed", "Not available" and "Not known" -- in the `entity` as well as `type` feature (containing 32 and 2 missing values, respectively). In addition, I must correct one errorneous fined entity (violation id 246). Finally, I clean the `violation` predictor using regex and the `stringr` package.

```r
gdpr_data <- gdpr_data %>% 
  rename(country = name, entity = controller, violation = articleViolated) %>% 
  mutate(across(date, ~na_if(lubridate::mdy(.), "1970-01-01"))) %>% 
  mutate(across(price, ~if_else(. == 0, NA_integer_, .))) %>% 
  mutate(across(c(entity, type), ~if_else(str_detect(., "Unknown|Not"), NA_character_, .))) %>% 
  mutate(across(entity, ~str_replace(., "https://datenschutz-hamburg.de/assets/pdf/28._Taetigkeitsbericht_Datenschutz_2019_HmbBfDI.pdf", "HVV GmbH"))) %>% 
  mutate(across(violation, ~if_else(is.na(str_extract(., ">.+<")), ., str_extract(., ">.+<") %>% str_sub(., 2, -2)))) %>%
  mutate(across(c(violation, authority), ~str_replace_all(., "\\t", "")))
```

Since a cross-check of the entity names reveals quite a few inconsistencies in how the entities have been written in the databse, I leverage the `stringdist` package [@vanderLoo.2014] for fuzzy string matching to homogenize some of the entries. For example, the *optimal string alignment* (*osa*) measure allows to assess the similarity of two strings by enumerating the number of pre-processing steps (deletion, insertion, substitution and transposition) necessary to transform one string into another. Adhering to the following assumptions yields four fuzzy matches which are accounted for in the subsequent EDA:
* Set the minimum-osa threshold to 3 (i.e. only consider string pairs which require three transformations to be aligned).
* Only consider strings of length > 3 (otherwise the minimum-osa threshold becomes redundant).

```
> # A tibble: 4 x 5
>   ent_a                          ent_b                           osa  id.a  id.b
>   <chr>                          <chr>                         <dbl> <int> <int>
> 1 Telecommunication Service Pro~ Telecommunication service pr~     2     7    48
> 2 A mayor                        Mayor                             3    32   100
> 3 A bank                         Bank                              3    50    55
> 4 Vodafone Espana                Vodafone España                   1    64   156
```


```r
gdpr_data <- gdpr_data %>% 
  mutate(across(entity, ~str_replace_all(., c("Telecommunication Service Provider" = "Telecommunication service provider",
                                              "A mayor" = "Mayor",
                                              "A bank" = "Bank",
                                              "Vodafone Espana" = "Vodafone España"))))
```

Finally, let's have a look at the cleaned data.

```
>    id        country price
> 1   6        Romania 20000
> 2  86 United Kingdom    NA
> 3 113          Spain   900
> 4  87 United Kingdom 80000
> 5 140        Hungary 15100
>                                                                                authority
> 1         Romanian National Supervisory Authority for Personal Data Processing (ANSPDCP)
> 2                                                               Information Commissioner
> 3                                               Spanish Data Protection Authority (AEPD)
> 4                                                               Information Commissioner
> 5 Hungarian National Authority for Data Protection and the Freedom of Information (NAIH)
>         date                                     entity
> 1 2019-10-09                           Vreau Credit SRL
> 2 2019-07-10 Driver and Vehicle Licensing Agency (DVLA)
> 3 2019-11-07                       TODOTECNICOS24H S.L.
> 4 2019-07-16                    Life at Parliament View
> 5 2019-10-01                            Town of Kerepes
>                    violation
> 1 Art. 32 GDPR, Art. 33 GDPR
> 2                    Unknown
> 3               Art. 13 GDPR
> 4   Data Protection Act 2018
> 5            Art. 6 (1) GDPR
>                                                                      type
> 1 Failure to implement sufficient measures to ensure information security
> 2                                            Non-compliance (Data Breach)
> 3                                   Information obligation non-compliance
> 4                                            Non-compliance (Data Breach)
> 5                    Non-compliance with lawful basis for data processing
>                                                                                                                                      source
> 1                                                                    https://www.dataprotection.ro/?page=Comunicat_Presa_09_10_2019&lang=ro
> 2              https://www.autoexpress.co.uk/car-news/consumer-news/91275/dvla-sale-of-driver-details-to-private-parking-firms-looked-at-by
> 3                                                                                    https://www.aepd.es/resoluciones/PS-00268-2019_ORI.pdf
> 4 https://ico.org.uk/about-the-ico/news-and-events/news-and-blogs/2019/07/estate-agency-fined-80-000-for-failing-to-keep-tenants-data-safe/
> 5                                                                                    https://www.naih.hu/files/NAIH-2019-2076-hatarozat.pdf
```

Now let's briefly validate the integrity of the scraped dataset.
<table class="table table-striped table-hover table-condensed table-responsive" style="margin-left: auto; margin-right: auto;">
<caption>TEEEEST 2</caption>
 <thead>
  <tr>
   <th style="text-align:right;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> id </th>
   <th style="text-align:right;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> picture </th>
   <th style="text-align:right;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> country </th>
   <th style="text-align:right;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> price </th>
   <th style="text-align:right;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> authority </th>
   <th style="text-align:right;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> date </th>
   <th style="text-align:right;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> entity </th>
   <th style="text-align:right;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> violation </th>
   <th style="text-align:right;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> type </th>
   <th style="text-align:right;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> source </th>
   <th style="text-align:right;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> summary </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 11 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 15 </td>
   <td style="text-align:right;"> 32 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 0 </td>
  </tr>
</tbody>
</table>
And indeed, a quick glance at the missing values per feature reveals numerous missing values for the `price` (4.03%), `date` (5.49%) and `entity` (11.72%) feature. Without diving deeper into the information sources, it may be assumed that for the affected court cases no complete record of the verdict was openly published by the jurisdiction.

Also, with regards to some of the fines, *PrivacyAffairs* explicitely states that "*The Marriott and British Airways cases are not final yet and the fines are just proposals. Other GDPR fines trackers incorrectly report those as final.*"

### Exploratory Data Analysis



First ever fine

```r
gdpr_data %>% 
  select(-summary, -picture) %>% 
  filter(date == min(date, na.rm = TRUE))
>   id  country price                                                authority
> 1 78 Bulgaria   500 Bulgarian Commission for Personal Data Protection (KZLD)
>         date entity                       violation
> 1 2018-05-12   Bank Art. 5 (1) b) GDPR, Art. 6 GDPR
>                                                   type
> 1 Non-compliance with lawful basis for data processing
>                                         source
> 1 https://www.cpdp.bg/?p=element_view&aid=2152
```

Top 10 fines

```r
gdpr_data %>%
  select(-summary, -picture) %>% 
  slice_max(order_by = price, n = 10) %>% 
  knitr::kable(
    caption = "TEEEEST 3",
  ) %>% 
  kable_styling(bootstrap_options = c("striped", "hover", "condensed", "responsive"), full_width = T) %>% 
  row_spec(0, bold = TRUE, color = "white", background = "#8486B2") %>% 
  column_spec(c(4,7,8,9), width = "10cm", background = "yellow")
```

<table class="table table-striped table-hover table-condensed table-responsive" style="margin-left: auto; margin-right: auto;">
<caption>TEEEEST 3</caption>
 <thead>
  <tr>
   <th style="text-align:right;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> id </th>
   <th style="text-align:left;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> country </th>
   <th style="text-align:right;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> price </th>
   <th style="text-align:left;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> authority </th>
   <th style="text-align:left;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> date </th>
   <th style="text-align:left;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> entity </th>
   <th style="text-align:left;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> violation </th>
   <th style="text-align:left;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> type </th>
   <th style="text-align:left;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> source </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> 66 </td>
   <td style="text-align:left;"> France </td>
   <td style="text-align:right;"> 50000000 </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> French Data Protection Authority (CNIL) </td>
   <td style="text-align:left;"> 2019-01-21 </td>
   <td style="text-align:left;"> Google Inc. </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Art. 13 GDPR, Art. 14 GDPR, Art. 6 GDPR, Art. 4 GDPR, Art. 5 GDPR </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Several </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> https://www.cnil.fr/en/cnils-restricted-committee-imposes-financial-penalty-50-million-euros-against-google-llc </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 200 </td>
   <td style="text-align:left;"> Italy </td>
   <td style="text-align:right;"> 27802946 </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Italian Data Protection Authority (Garante) </td>
   <td style="text-align:left;"> 2020-02-01 </td>
   <td style="text-align:left;"> TIM - Telecom Provider </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Art. 58(2) GDPR </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Non-cooperation with Data Protection Authority </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> https://www.garanteprivacy.it/web/guest/home/docweb/-/docweb-display/docweb/9256409 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 79 </td>
   <td style="text-align:left;"> Austria </td>
   <td style="text-align:right;"> 18000000 </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Austrian Data Protection Authority (DSB) </td>
   <td style="text-align:left;"> 2019-10-23 </td>
   <td style="text-align:left;"> Austrian Post </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Art. 5 (1) a) GDPR, Art. 6 GDPR </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Non-compliance with lawful basis for data processing </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> https://wien.orf.at/stories/3019396/ </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 82 </td>
   <td style="text-align:left;"> Germany </td>
   <td style="text-align:right;"> 14500000 </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Data Protection Authority of Baden-Wuerttemberg </td>
   <td style="text-align:left;"> 2019-10-30 </td>
   <td style="text-align:left;"> Deutsche Wohnen SE </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Art. 5 GDPR, Art. 25 GDPR </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Failure to comply with data processing principles </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> https://www.lexology.com/library/detail.aspx?g=1e75e1a5-2bb6-409c-b1dd-239f51bdb2bd </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 138 </td>
   <td style="text-align:left;"> Germany </td>
   <td style="text-align:right;"> 9550000 </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> The Federal Commissioner for Data Protection and Freedom of Information (BfDI) </td>
   <td style="text-align:left;"> 2019-12-09 </td>
   <td style="text-align:left;"> 1&amp;1 Telecom GmbH </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Art. 32 GDPR </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Failure to implement sufficient measures to ensure information security </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> https://www.bfdi.bund.de/DE/Infothek/Pressemitteilungen/2019/30_BfDIverh%C3%A4ngtGeldbu%C3%9Fe1u1.html </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 189 </td>
   <td style="text-align:left;"> Italy </td>
   <td style="text-align:right;"> 8500000 </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Italian Data Protection Authority (Garante) </td>
   <td style="text-align:left;"> 2020-01-17 </td>
   <td style="text-align:left;"> Eni Gas e Luce </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Art. 5 GDPR, Art. 6 GDPR, Art. 17 GDPR, Art. 21 GDPR </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Non-compliance with lawful basis for data processing </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> https://www.gpdp.it/web/guest/home/docweb/-/docweb-display/docweb/9244365 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 237 </td>
   <td style="text-align:left;"> Sweden </td>
   <td style="text-align:right;"> 7000000 </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Data Protection Authority of Sweden </td>
   <td style="text-align:left;"> 2020-03-11 </td>
   <td style="text-align:left;"> Google </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Art. 5 GDPR, Art. 6 GDPR, Art. 17 GDPR </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Failure to comply with data processing principles </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> https://www.datainspektionen.se/globalassets/dokument/beslut/2020-03-11-beslut-google.pdf </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 190 </td>
   <td style="text-align:left;"> Italy </td>
   <td style="text-align:right;"> 3000000 </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Italian Data Protection Authority (Garante) </td>
   <td style="text-align:left;"> 2020-01-17 </td>
   <td style="text-align:left;"> Eni Gas e Luce </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Art. 5 GDPR, Art. 6 GDPR </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Non-compliance with lawful basis for data processing </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> https://www.gpdp.it/web/guest/home/docweb/-/docweb-display/docweb/9244365 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 15 </td>
   <td style="text-align:left;"> Bulgaria </td>
   <td style="text-align:right;"> 2600000 </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Data Protection Commission of Bulgaria (KZLD) </td>
   <td style="text-align:left;"> 2019-08-28 </td>
   <td style="text-align:left;"> National Revenue Agency </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Art. 32 GDPR </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Failure to implement sufficient measures to ensure information security </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> https://www.cpdp.bg/index.php?p=news_view&amp;aid=1519 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 108 </td>
   <td style="text-align:left;"> Netherlands </td>
   <td style="text-align:right;"> 900000 </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Dutch Supervisory Authority for Data Protection (AP) </td>
   <td style="text-align:left;"> 2019-10-31 </td>
   <td style="text-align:left;"> UWV - Insurance provider </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Art. 32 GDPR </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> Failure to implement sufficient measures to ensure information security </td>
   <td style="text-align:left;width: 10cm; background-color: yellow !important;"> https://autoriteitpersoonsgegevens.nl/nl/nieuws/ap-dwingt-uwv-met-sanctie-gegevens-beter-te-beveiligen </td>
  </tr>
</tbody>
</table>

Plot1
<img src="index_files/figure-html/plot1-1.png" width="70%" style="display: block; margin: auto;" />

Art. 83 GDPR: 

https://gdpr.eu/article-83-conditions-for-imposing-administrative-fines/

Nr. 4: 10m€ or 2% of total revenue
Nr. 5: 20m€ or 4% of total revenue

Plot2
<img src="index_files/figure-html/plot2-1.png" width="70%" style="display: block; margin: auto;" />

Plot3
<img src="index_files/figure-html/plot3-1.png" width="70%" style="display: block; margin: auto;" />

#which article was most frequently violated?
- drop all fines for which no GDPR article is stated
- on the country side we see a most diverse set of violated articles on the side of romania and hungary
- poland and croatia, almost no activity

Plot4
<img src="index_files/figure-html/plot4-1.png" width="70%" style="display: block; margin: auto;" />


Article that incurred the highes average fine
Compute average fine by dividng by the number of articles involved in the fine

Plot5
<img src="index_files/figure-html/plot5-1.png" width="70%" style="display: block; margin: auto;" />

Plot6
<img src="index_files/figure-html/plot6-1.png" width="70%" style="display: block; margin: auto;" />

### References
