---
title: "GDPR Fines - Exploratory Data Analysis 22"
date: 2020-06-11
categories:
  - rstats
  - tidymodels
tags:
  - rstats
  - tidymodels
subtitle: ''
summary: "Use tidymodels for unsupervised dimensionality reduction."
featured: no
image:
  caption: 'Testing'
  focal_point: ''
  preview_only: no
projects: []
bibliography: references.bib
csl: "https://raw.githubusercontent.com/citation-style-language/styles/master/apa.csl"
---

Data Privacy and Data Security -- two espoused concepts that gain substantial traction in the current times of Covid-19, especially in the context of the upcoming German Corona tracking app [@SAP.2020] that is about to be released next week. Public expectations compete with individual reservations and data privacy concerns, partly fueled by events such as the 2013 Snowden revelations or the 2018 Camebridge Analytica scandal. Likewise, events like these contributed to the willingness of the EU to enforce supranational regulations on data privacy and security, climaxing the in the EU-GDPR [@EuropeanUnion.2018] which is binding for all EU member states since 2018-05-25.

In this blog post, I would like to take the current debate as an opportunity to dive into a dataset about monetary fines since the issuance of the EU-GDPR. In the process of doing so, I will do some web scraping to extract the dataset from the *GDPR Fines Tracker* on *privacyaffairs.com* [@PrivacyAffairs.2020], clean the dataset using some of the tools recently released as part of the `dplyr v1.0.0` release [@Wickham.2020], experiment with some fuzzy string matching capabilities provided in the `stringdist` package [@vanderLoo.2014] and engage in some visual EDA. Meanwhile, this post is inspired by [David Robinson](https://www.youtube.com/watch?v=EVvnnWKO_4w)'s TidyTuesday Series. The whole analysis is embeeded into an R markdown document.




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

<table class="table table-striped table-hover table-condensed table-responsive" style="margin-left: auto; margin-right: auto;">
<caption>TEEEEEEEEESSSSST</caption>
 <thead>
  <tr>
   <th style="text-align:right;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> id </th>
   <th style="text-align:left;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> name </th>
   <th style="text-align:right;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> price </th>
   <th style="text-align:left;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> authority </th>
   <th style="text-align:left;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> date </th>
   <th style="text-align:left;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> controller </th>
   <th style="text-align:left;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> articleViolated </th>
   <th style="text-align:left;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> type </th>
   <th style="text-align:left;font-weight: bold;color: white !important;background-color: #8486B2 !important;"> source </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:left;"> Romania </td>
   <td style="text-align:right;"> 2,500 </td>
   <td style="text-align:left;"> Romanian National Supervisory Authority for Personal Data Processing (ANSPDCP) </td>
   <td style="text-align:left;"> 10/17/2019 </td>
   <td style="text-align:left;"> UTTIS INDUSTRIES </td>
   <td style="text-align:left;"> Art. 12 GDPR, Art. 13 GDPR, Art. 5 (1) c) GDPR, Art. 6 GDPR </td>
   <td style="text-align:left;"> Information obligation non-compliance </td>
   <td style="text-align:left;"> https://www.dataprotection.ro/?page=A_patra_amenda&amp;lang=ro </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 108 </td>
   <td style="text-align:left;"> Netherlands </td>
   <td style="text-align:right;"> 900,000 </td>
   <td style="text-align:left;"> Dutch Supervisory Authority for Data Protection (AP) </td>
   <td style="text-align:left;"> 10/31/2019 </td>
   <td style="text-align:left;"> UWV - Insurance provider </td>
   <td style="text-align:left;"> Art. 32 GDPR </td>
   <td style="text-align:left;"> Failure to implement sufficient measures to ensure information security </td>
   <td style="text-align:left;"> https://autoriteitpersoonsgegevens.nl/nl/nieuws/ap-dwingt-uwv-met-sanctie-gegevens-beter-te-beveiligen </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 199 </td>
   <td style="text-align:left;"> Spain </td>
   <td style="text-align:right;"> 10,000 </td>
   <td style="text-align:left;"> Spanish Data Protection Authority (AEPD) </td>
   <td style="text-align:left;"> 01/07/2020 </td>
   <td style="text-align:left;"> Asociación de Médicos Demócratas </td>
   <td style="text-align:left;"> Art. 6 GDPR </td>
   <td style="text-align:left;"> Non-compliance with lawful basis for data processing </td>
   <td style="text-align:left;"> https://www.aepd.es/es/documento/ps-00231-2019.pdf </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 39 </td>
   <td style="text-align:left;"> Czech Republic </td>
   <td style="text-align:right;"> 194 </td>
   <td style="text-align:left;"> Czech Data Protection Auhtority (UOOU) </td>
   <td style="text-align:left;"> 05/06/2019 </td>
   <td style="text-align:left;"> Public utility company </td>
   <td style="text-align:left;"> Art. 15 GDPR </td>
   <td style="text-align:left;"> Non-compliance with lawful basis for data processing </td>
   <td style="text-align:left;"> https://www.uoou.cz/assets/File.ashx?id_org=200144&amp;id_dokumenty=34472 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 34 </td>
   <td style="text-align:left;"> France </td>
   <td style="text-align:right;"> 400,000 </td>
   <td style="text-align:left;"> French Data Protection Authority (CNIL) </td>
   <td style="text-align:left;"> 05/28/2019 </td>
   <td style="text-align:left;"> SERGIC </td>
   <td style="text-align:left;"> Art. 32 GDPR </td>
   <td style="text-align:left;"> Failure to implement sufficient measures to ensure information security </td>
   <td style="text-align:left;"> https://www.legifrance.gouv.fr/affichCnil.do?oldAction=rechExpCnil&amp;id=CNILTEXT000038552658&amp;fastReqId=119744754&amp;fastPos=1 </td>
  </tr>
</tbody>
</table>

### Data Cleaning

In a next step, I streamline the dataset a little more and try to get rid of the various inconsistencies in the data entries. 

First, I would like to adjust some of the column names to highlight the actual content of the features. I rename `name` to `country`, `controller` to `entity` (i.e. the entity fined as a result of the violation) and `articleViolated` to `violation`. Second, I infer from the sample above that the violation `date` is not in standard international date format (`jjjj-mm-dd`). Let's change this using the `lubridate` package while properly accounting for 15 `NA`s (indicated by unix time `1970-01-01`). Third, I format the `price` feature by specyfing it as a proper currency (using the `scales` package). Fourth, and analogue to the violation `date`, I properly account for `NA`s -- taking the form "Not disclosed", "Not available" and "Not known" -- in the `entity` as well as `type` feature (containing 32 and 2 missing values, respectively). In addition, I must correct one errorneous fined entity (violation id 246). Finally, I clean the `violation` predictor using regex and the `stringr` package.

```r
gdpr_data <- gdpr_data %>% 
  #1
  rename(country = name, entity = controller, violation = articleViolated) %>% 
  #2
  mutate(across(date, ~na_if(lubridate::mdy(.), "1970-01-01"))) %>% 
  #3
  mutate(across(price, ~if_else(. == 0, NA_integer_, .))) %>% 
  #4
  mutate(across(c(entity, type), ~if_else(str_detect(., "Unknown|Not"), NA_character_, .))) %>% 
  mutate(across(entity, ~str_replace(., "https://datenschutz-hamburg.de/assets/pdf/28._Taetigkeitsbericht_Datenschutz_2019_HmbBfDI.pdf", "HVV GmbH"))) %>% 
  #5
  mutate(across(violation, ~if_else(is.na(str_extract(., ">.+<")), ., str_extract(., ">.+<") %>% str_sub(., 2, -2)))) %>%
  mutate(across(c(violation, authority), ~str_replace_all(., "\\t", "")))
```

Since a cross-check of the entity names reveals quite a few inconsistencies in how the entities have been written in the databse, I leverage the `stringdist` package [@vanderLoo.2014] for fuzzy string matching to homogenize some of the entries. For example, the *optimal string alignment* (*osa*) measure allows to assess the similarity of two strings by enumerating the number of pre-processing steps (deletion, insertion, substitution and transposition) necessary to transform one string into another. Adhering to the following assumptions yields four fuzzy matches which are accounted for in the subsequent EDA:
* Set the minimum-osa threshold to 3 (i.e. only consider string pairs which require three transformations to be aligned).
* Only consider strings of length > 3 (otherwise the minimum-osa threshold becomes redundant).

```r
entities <- gdpr_data %>% 
  distinct(entity) %>% 
  drop_na %>% 
  mutate(id = row_number(), .before = 1)


fuzzy_matches <- unique(gdpr_data$entity[!is.na(gdpr_data$entity)]) %>% 
  expand_grid(ent_a = ., ent_b = .) %>% 
  mutate(osa = stringdist(ent_a, ent_b, method = "dl", nthread = 4)) %>% 
  filter(osa < 4L &
           osa != 0L &
           str_length(ent_a) > 3L &
           str_length(ent_b) > 3L) %>% 
  left_join(entities, by = c("ent_a" = "entity"), suffix = c(".a", ".b")) %>% 
  left_join(entities, by = c("ent_b" = "entity"), suffix = c(".a", ".b")) %>% 
  filter(id.a < id.b)

fuzzy_matches
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["ent_a"],"name":[1],"type":["chr"],"align":["left"]},{"label":["ent_b"],"name":[2],"type":["chr"],"align":["left"]},{"label":["osa"],"name":[3],"type":["dbl"],"align":["right"]},{"label":["id.a"],"name":[4],"type":["int"],"align":["right"]},{"label":["id.b"],"name":[5],"type":["int"],"align":["right"]}],"data":[{"1":"Telecommunication Service Provider","2":"Telecommunication service provider","3":"2","4":"7","5":"48"},{"1":"A mayor","2":"Mayor","3":"3","4":"32","5":"100"},{"1":"A bank","2":"Bank","3":"3","4":"50","5":"55"},{"1":"Vodafone Espana","2":"Vodafone España","3":"1","4":"64","5":"156"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

```r

gdpr_data <- gdpr_data %>% 
  mutate(across(entity, ~str_replace_all(., c("Telecommunication Service Provider" = "Telecommunication service provider",
                                              "A mayor" = "Mayor",
                                              "A bank" = "Bank",
                                              "Vodafone Espana" = "Vodafone España"))))
```

Finally, let's have a look at the cleaned data.
<table class="table table-striped table-hover table-condensed table-responsive" style="margin-left: auto; margin-right: auto;">
<caption>TEEEEEEEEEEST</caption>
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
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:left;"> Romania </td>
   <td style="text-align:right;"> 2,500 </td>
   <td style="text-align:left;"> Romanian National Supervisory Authority for Personal Data Processing (ANSPDCP) </td>
   <td style="text-align:left;"> 2019-10-17 </td>
   <td style="text-align:left;"> UTTIS INDUSTRIES </td>
   <td style="text-align:left;"> Art. 12 GDPR, Art. 13 GDPR, Art. 5 (1) c) GDPR, Art. 6 GDPR </td>
   <td style="text-align:left;"> Information obligation non-compliance </td>
   <td style="text-align:left;"> https://www.dataprotection.ro/?page=A_patra_amenda&amp;lang=ro </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 108 </td>
   <td style="text-align:left;"> Netherlands </td>
   <td style="text-align:right;"> 900,000 </td>
   <td style="text-align:left;"> Dutch Supervisory Authority for Data Protection (AP) </td>
   <td style="text-align:left;"> 2019-10-31 </td>
   <td style="text-align:left;"> UWV - Insurance provider </td>
   <td style="text-align:left;"> Art. 32 GDPR </td>
   <td style="text-align:left;"> Failure to implement sufficient measures to ensure information security </td>
   <td style="text-align:left;"> https://autoriteitpersoonsgegevens.nl/nl/nieuws/ap-dwingt-uwv-met-sanctie-gegevens-beter-te-beveiligen </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 199 </td>
   <td style="text-align:left;"> Spain </td>
   <td style="text-align:right;"> 10,000 </td>
   <td style="text-align:left;"> Spanish Data Protection Authority (AEPD) </td>
   <td style="text-align:left;"> 2020-01-07 </td>
   <td style="text-align:left;"> Asociación de Médicos Demócratas </td>
   <td style="text-align:left;"> Art. 6 GDPR </td>
   <td style="text-align:left;"> Non-compliance with lawful basis for data processing </td>
   <td style="text-align:left;"> https://www.aepd.es/es/documento/ps-00231-2019.pdf </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 39 </td>
   <td style="text-align:left;"> Czech Republic </td>
   <td style="text-align:right;"> 194 </td>
   <td style="text-align:left;"> Czech Data Protection Auhtority (UOOU) </td>
   <td style="text-align:left;"> 2019-05-06 </td>
   <td style="text-align:left;"> Public utility company </td>
   <td style="text-align:left;"> Art. 15 GDPR </td>
   <td style="text-align:left;"> Non-compliance with lawful basis for data processing </td>
   <td style="text-align:left;"> https://www.uoou.cz/assets/File.ashx?id_org=200144&amp;id_dokumenty=34472 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 34 </td>
   <td style="text-align:left;"> France </td>
   <td style="text-align:right;"> 400,000 </td>
   <td style="text-align:left;"> French Data Protection Authority (CNIL) </td>
   <td style="text-align:left;"> 2019-05-28 </td>
   <td style="text-align:left;"> SERGIC </td>
   <td style="text-align:left;"> Art. 32 GDPR </td>
   <td style="text-align:left;"> Failure to implement sufficient measures to ensure information security </td>
   <td style="text-align:left;"> https://www.legifrance.gouv.fr/affichCnil.do?oldAction=rechExpCnil&amp;id=CNILTEXT000038552658&amp;fastReqId=119744754&amp;fastPos=1 </td>
  </tr>
</tbody>
</table>

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
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["id"],"name":[1],"type":["int"],"align":["right"]},{"label":["country"],"name":[2],"type":["chr"],"align":["left"]},{"label":["price"],"name":[3],"type":["int"],"align":["right"]},{"label":["authority"],"name":[4],"type":["chr"],"align":["left"]},{"label":["date"],"name":[5],"type":["date"],"align":["right"]},{"label":["entity"],"name":[6],"type":["chr"],"align":["left"]},{"label":["violation"],"name":[7],"type":["chr"],"align":["left"]},{"label":["type"],"name":[8],"type":["chr"],"align":["left"]},{"label":["source"],"name":[9],"type":["chr"],"align":["left"]}],"data":[{"1":"78","2":"Bulgaria","3":"500","4":"Bulgarian Commission for Personal Data Protection (KZLD)","5":"2018-05-12","6":"Bank","7":"Art. 5 (1) b) GDPR, Art. 6 GDPR","8":"Non-compliance with lawful basis for data processing","9":"https://www.cpdp.bg/?p=element_view&aid=2152"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

Top 10 fines

```r
gdpr_data %>%
  select(-summary, -picture) %>% 
  slice_max(order_by = price, n = 10) %>% 
  knitr::kable(
    caption = "TEEEEST 3",
  ) %>% 
  kable_styling(bootstrap_options = c("striped", "hover", "condensed", "responsive"), full_width = T) %>% 
  row_spec(0, bold = TRUE, color = "white", background = "#8486B2")
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
   <td style="text-align:left;"> French Data Protection Authority (CNIL) </td>
   <td style="text-align:left;"> 2019-01-21 </td>
   <td style="text-align:left;"> Google Inc. </td>
   <td style="text-align:left;"> Art. 13 GDPR, Art. 14 GDPR, Art. 6 GDPR, Art. 4 GDPR, Art. 5 GDPR </td>
   <td style="text-align:left;"> Several </td>
   <td style="text-align:left;"> https://www.cnil.fr/en/cnils-restricted-committee-imposes-financial-penalty-50-million-euros-against-google-llc </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 200 </td>
   <td style="text-align:left;"> Italy </td>
   <td style="text-align:right;"> 27802946 </td>
   <td style="text-align:left;"> Italian Data Protection Authority (Garante) </td>
   <td style="text-align:left;"> 2020-02-01 </td>
   <td style="text-align:left;"> TIM - Telecom Provider </td>
   <td style="text-align:left;"> Art. 58(2) GDPR </td>
   <td style="text-align:left;"> Non-cooperation with Data Protection Authority </td>
   <td style="text-align:left;"> https://www.garanteprivacy.it/web/guest/home/docweb/-/docweb-display/docweb/9256409 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 79 </td>
   <td style="text-align:left;"> Austria </td>
   <td style="text-align:right;"> 18000000 </td>
   <td style="text-align:left;"> Austrian Data Protection Authority (DSB) </td>
   <td style="text-align:left;"> 2019-10-23 </td>
   <td style="text-align:left;"> Austrian Post </td>
   <td style="text-align:left;"> Art. 5 (1) a) GDPR, Art. 6 GDPR </td>
   <td style="text-align:left;"> Non-compliance with lawful basis for data processing </td>
   <td style="text-align:left;"> https://wien.orf.at/stories/3019396/ </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 82 </td>
   <td style="text-align:left;"> Germany </td>
   <td style="text-align:right;"> 14500000 </td>
   <td style="text-align:left;"> Data Protection Authority of Baden-Wuerttemberg </td>
   <td style="text-align:left;"> 2019-10-30 </td>
   <td style="text-align:left;"> Deutsche Wohnen SE </td>
   <td style="text-align:left;"> Art. 5 GDPR, Art. 25 GDPR </td>
   <td style="text-align:left;"> Failure to comply with data processing principles </td>
   <td style="text-align:left;"> https://www.lexology.com/library/detail.aspx?g=1e75e1a5-2bb6-409c-b1dd-239f51bdb2bd </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 138 </td>
   <td style="text-align:left;"> Germany </td>
   <td style="text-align:right;"> 9550000 </td>
   <td style="text-align:left;"> The Federal Commissioner for Data Protection and Freedom of Information (BfDI) </td>
   <td style="text-align:left;"> 2019-12-09 </td>
   <td style="text-align:left;"> 1&amp;1 Telecom GmbH </td>
   <td style="text-align:left;"> Art. 32 GDPR </td>
   <td style="text-align:left;"> Failure to implement sufficient measures to ensure information security </td>
   <td style="text-align:left;"> https://www.bfdi.bund.de/DE/Infothek/Pressemitteilungen/2019/30_BfDIverh%C3%A4ngtGeldbu%C3%9Fe1u1.html </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 189 </td>
   <td style="text-align:left;"> Italy </td>
   <td style="text-align:right;"> 8500000 </td>
   <td style="text-align:left;"> Italian Data Protection Authority (Garante) </td>
   <td style="text-align:left;"> 2020-01-17 </td>
   <td style="text-align:left;"> Eni Gas e Luce </td>
   <td style="text-align:left;"> Art. 5 GDPR, Art. 6 GDPR, Art. 17 GDPR, Art. 21 GDPR </td>
   <td style="text-align:left;"> Non-compliance with lawful basis for data processing </td>
   <td style="text-align:left;"> https://www.gpdp.it/web/guest/home/docweb/-/docweb-display/docweb/9244365 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 237 </td>
   <td style="text-align:left;"> Sweden </td>
   <td style="text-align:right;"> 7000000 </td>
   <td style="text-align:left;"> Data Protection Authority of Sweden </td>
   <td style="text-align:left;"> 2020-03-11 </td>
   <td style="text-align:left;"> Google </td>
   <td style="text-align:left;"> Art. 5 GDPR, Art. 6 GDPR, Art. 17 GDPR </td>
   <td style="text-align:left;"> Failure to comply with data processing principles </td>
   <td style="text-align:left;"> https://www.datainspektionen.se/globalassets/dokument/beslut/2020-03-11-beslut-google.pdf </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 190 </td>
   <td style="text-align:left;"> Italy </td>
   <td style="text-align:right;"> 3000000 </td>
   <td style="text-align:left;"> Italian Data Protection Authority (Garante) </td>
   <td style="text-align:left;"> 2020-01-17 </td>
   <td style="text-align:left;"> Eni Gas e Luce </td>
   <td style="text-align:left;"> Art. 5 GDPR, Art. 6 GDPR </td>
   <td style="text-align:left;"> Non-compliance with lawful basis for data processing </td>
   <td style="text-align:left;"> https://www.gpdp.it/web/guest/home/docweb/-/docweb-display/docweb/9244365 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 15 </td>
   <td style="text-align:left;"> Bulgaria </td>
   <td style="text-align:right;"> 2600000 </td>
   <td style="text-align:left;"> Data Protection Commission of Bulgaria (KZLD) </td>
   <td style="text-align:left;"> 2019-08-28 </td>
   <td style="text-align:left;"> National Revenue Agency </td>
   <td style="text-align:left;"> Art. 32 GDPR </td>
   <td style="text-align:left;"> Failure to implement sufficient measures to ensure information security </td>
   <td style="text-align:left;"> https://www.cpdp.bg/index.php?p=news_view&amp;aid=1519 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 108 </td>
   <td style="text-align:left;"> Netherlands </td>
   <td style="text-align:right;"> 900000 </td>
   <td style="text-align:left;"> Dutch Supervisory Authority for Data Protection (AP) </td>
   <td style="text-align:left;"> 2019-10-31 </td>
   <td style="text-align:left;"> UWV - Insurance provider </td>
   <td style="text-align:left;"> Art. 32 GDPR </td>
   <td style="text-align:left;"> Failure to implement sufficient measures to ensure information security </td>
   <td style="text-align:left;"> https://autoriteitpersoonsgegevens.nl/nl/nieuws/ap-dwingt-uwv-met-sanctie-gegevens-beter-te-beveiligen </td>
  </tr>
</tbody>
</table>

Top 10 fined entities

```r
euro <- scales::dollar_format(
  prefix = "",
  suffix = "",
  big.mark = ",",
  decimal.mark = "."
)

gdpr_data %>%
  drop_na(price, date) %>% 
  mutate_at(vars(entity), ~as.factor(.) %>% 
            fct_lump_n(., 8, w = price) %>% 
            fct_explicit_na(., na_level = "Other")) %>%
  
  
  ggplot(., aes(date, price / 1000000)) +
    scale_y_log10(labels = euro) +
    scale_x_date(date_breaks = "3 month",
                 date_minor_breaks = "1 month",
                 date_labels = "%Y-%m",
                 limits = c(as.Date("2018-05-01"), NA)) +
    geom_point(aes(color = entity)) +
    geom_point(size = 3, shape = 1, data = . %>% filter(entity != "Other")) +
    scale_color_brewer(palette = "Set3") +
    labs(
      x = "",
      y = "Imposed Fine [log m€]",
      title = "Distribution of Fines since 2018-05-28",
      subtitle = "Few very Heafty Fines Dominate the Landscape",
      caption = "Data from privacyaffairs.com"
    ) +
    theme_classic() +
    theme(
      text = element_text(family = "gg_font"),
      plot.title = element_text(size = font_size_title, face = "bold"),
      plot.subtitle = element_text(size = font_size_subtitle),
      plot.caption = element_text(size = font_size_caption, face = "italic"),
      axis.text = element_text(size = font_size_other, color = "black"),
      axis.title = element_text(size = font_size_other),
      legend.position = "bottom",
      legend.title = element_blank()) +
    guides(
      color = guide_legend(
        nrow = 3,
        override.aes = list(size = 4)
      )
    )
```

<img src="index_files/figure-html/top10-1.png" width="70%" style="display: block; margin: auto;" />

Art. 83 GDPR: 

https://gdpr.eu/article-83-conditions-for-imposing-administrative-fines/

Nr. 4: 10m€ or 2% of total revenue
Nr. 5: 20m€ or 4% of total revenue

```r
gdpr_data %>%
  drop_na(price) %>% 
  mutate_at(vars(entity), ~as.factor(.) %>% 
              fct_lump_n(., 8, w = price) %>% 
              fct_explicit_na(., na_level = "Other")) %>% 
  group_by(entity) %>% 
  summarise(total_fine = sum(price),
            fine_freq = n()) %>% 
  
  
  ggplot(., aes(total_fine / 1000000, fct_reorder(entity, total_fine))) +
    geom_col(fill = signature_color) +
    geom_text(aes(label = euro(total_fine / 1000000)),
              size = font_size_other * .35, family = "gg_font", fontface = "bold",
              nudge_x = 2.5) +
    geom_vline(xintercept = 10, linetype = "dashed") +
    geom_vline(xintercept = 20, linetype = "dashed") +
    scale_x_continuous(labels = euro, breaks = seq(0, 50, 10)) +
    labs(
      x = "Imposed Fine [m€]",
      y = "",
      title = "Top 8 Fined Entites",
      subtitle = "10m€ Threshold only Cracked by Five Individual Entities",
      caption = "Data from privacyaffairs.com"
    ) +
    theme_classic() +
    theme(
      text = element_text(family = "gg_font"),
      plot.title = element_text(size = font_size_title, face = "bold"),
      plot.subtitle = element_text(size = font_size_subtitle),
      plot.caption = element_text(size = font_size_caption, face = "italic"),
      axis.text = element_text(size = font_size_other, color = "black"),
      axis.title = element_text(size = font_size_other)
    )
> `summarise()` ungrouping output (override with `.groups` argument)
```

<img src="index_files/figure-html/unnamed-chunk-3-1.png" width="70%" style="display: block; margin: auto;" />



```r
gdpr_data %>%
  drop_na(price) %>% 
  mutate_at(vars(country), ~as.factor(.) %>% 
              fct_lump_n(., 12, w = price) %>% 
              fct_explicit_na(., na_level = "Other")) %>% 
  group_by(country) %>% 
  summarise(total_fine = sum(price),
            fine_freq = n(),
            .groups = "rowwise") %>% 

  ggplot(aes(fct_reorder(str_wrap(country, 10), -total_fine), total_fine / 1000000)) +
    geom_col(fill = signature_color) +
    geom_text(aes(label = euro(total_fine / 1000000)),
              size = font_size_other * .35, family = "gg_font", fontface = "bold",
              nudge_y = 3) +
    scale_y_continuous(labels = euro, breaks = seq(0, 50, 10)) +
    labs(
      x = "Imposed Fine [m€]",
      y = "",
      title = "European Countries Ranked by Imposed Fines",
      subtitle = "The German 'Tin Foil' Lags Behind",
      caption = "Data from privacyaffairs.com"
    ) +
    theme_classic() +
    theme(
      text = element_text(family = "gg_font"),
      plot.title = element_text(size = font_size_title, face = "bold"),
      plot.subtitle = element_text(size = font_size_subtitle),
      plot.caption = element_text(size = font_size_caption, face = "italic"),
      axis.text = element_text(size = font_size_other, color = "black"),
      axis.text.x = element_text(angle = 90, hjust = 0.95,vjust = 0.2),
      axis.title = element_text(size = font_size_other)
    )
```

<img src="index_files/figure-html/unnamed-chunk-4-1.png" width="70%" style="display: block; margin: auto;" />

#which article was most frequently violated?
- drop all fines for which no GDPR article is stated
- on the country side we see a most diverse set of violated articles on the side of romania and hungary
- poland and croatia, almost no activity

```r


gdpr_data %>% 
  select(country, violation) %>% 
  separate_rows(violation, sep = ",") %>% 
  filter(str_detect(violation, "Art")) %>% 
  mutate(across(violation, ~str_extract(., "(?<=Art.\\s?)[0-9]+"))) %>%
  mutate(across(everything(), as.factor)) %>% 
  group_by(country, violation) %>% 
  summarise(freq = n(), .groups = "rowwise") %>% 
  
  
  ggplot(aes(
    x = fct_relevel(violation, function(x){as.character(sort(as.integer(x)))}),
    y = country
  )) +
    geom_point(aes(color = freq), shape = 15, size = 4) +
    scale_color_gradient(name = "Number\nof Fines", low = "#cdd8ff", high = "#021868") +
    labs(
      x = "GDPR Article Number",
      y = "",
      title = "Distribution of Violated Articles across Europe",
      subtitle = "Art. 5, 6, 32 GDPR as Major Stumbling Blocks"
    ) +
    theme_classic() +
    theme(
      text = element_text(family = "gg_font"),
      plot.title = element_text(size = font_size_title, face = "bold"),
      plot.subtitle = element_text(size = font_size_subtitle),
      plot.caption = element_text(size = font_size_caption, face = "italic"),
      axis.text = element_text(size = font_size_other, color = "black"),
      axis.title = element_text(size = font_size_other)
    )
```

<img src="index_files/figure-html/unnamed-chunk-5-1.png" width="70%" style="display: block; margin: auto;" />


Article that incurred the highes average fine
Compute average fine by dividng by the number of articles involved in the fine

```r
gdpr_data %>% 
  drop_na(price) %>% 
  select(violation, price) %>% 
  mutate(across(price, ~ . / str_count(violation, "Art"))) %>% 
  separate_rows(violation, sep = ",") %>% 
  filter(str_detect(violation, "Art")) %>% 
  mutate(across(violation, ~str_extract(., "(?<=Art.\\s?)[0-9]+") %>% as.factor %>% fct_lump(., 8, w = price))) %>%
  group_by(violation) %>% 
  summarise(avg_fine = sum(price),
            .groups = "rowwise") %>% 
  ungroup %>% 
  left_join(read_delim("./article_titles.txt", delim = "; ", col_types = cols("f", "c")), by = c("violation" = "article")) %>% 
  mutate(across(title, ~paste(violation, "-", .))) %>% 
  
  
  ggplot(aes(avg_fine / 1000000,
             title %>% str_trunc(50) %>% str_wrap(30) %>% as.factor %>% fct_reorder(., avg_fine, .desc = F))) +
    geom_col(fill = signature_color) +
    scale_x_continuous(labels = euro) +
    geom_vline(aes(xintercept = mean(avg_fine / 1000000)), linetype = "dashed") +
    labs(
      x = "Imposed Fine [m€]",
      y = "GDPR Article Number",
      title = "Average Fine per Violated Article",
      subtitle = "Art. 5, 6, 32 GDPR as Major Stumbling Blocks",
      caption = "Data from privacyaffairs.com"
    ) +
    theme_classic() +
    theme(
      text = element_text(family = "gg_font"),
      plot.title = element_text(size = font_size_title, face = "bold"),
      plot.subtitle = element_text(size = font_size_subtitle),
      plot.caption = element_text(size = font_size_caption, face = "italic"),
      axis.text = element_text(size = font_size_other, color = "black"),
      axis.title = element_text(size = font_size_other)
    )
```

<img src="index_files/figure-html/unnamed-chunk-6-1.png" width="70%" style="display: block; margin: auto;" />


```r
gdpr_data %>%
  drop_na(price, date) %>% 
  mutate(month = lubridate::month(date, label = T)) %>% 
  group_by(month) %>% 
  summarise(total_fine = sum(price),
            freq = n(),
            .groups = "rowwise") %>% 

  ggplot(aes(month, freq)) +
    geom_col(aes(fill = freq), width = 1, color = "white") +
    scale_y_continuous(limits = c(NA, 45)) +
    scale_fill_gradient(name = "Number\nof Fines", low = "#cdd8ff", high = "#021868")+
    coord_polar("x") +
    labs(
      x = "",
      y = "",
      title = "Yearly Ditribution of Fines",
      subtitle = "Hello Summer Slump - We're on Vacation",
      caption = "Data from privacyaffairs.com"
    ) +
    theme_classic() +
    theme(
      text = element_text(family = "gg_font"),
      plot.title = element_text(size = font_size_title, face = "bold"),
      plot.subtitle = element_text(size = font_size_subtitle),
      plot.caption = element_text(size = font_size_caption, face = "italic"),
      axis.text.x = element_text(size = font_size_caption),
      axis.text.y = element_blank(),
      axis.title = element_text(size = font_size_other),
      line = element_blank(),
      legend.position = "right"
    )
```

<img src="index_files/figure-html/unnamed-chunk-7-1.png" width="70%" style="display: block; margin: auto;" />

### References
