---
title: "Exploring GDPR Fines - The Tidy Approach"
author: Simon Schölzel
date: '2020-06-11'
slug: gdpr-violations
output:
  html_document:
    keep_md: true
categories:
- Rmarkdown
- rvest
- stringdist
- ggplot2
tags:
- Rmarkdown
- rvest
- stringdist
- ggplot2
subtitle: ''
summary: 'Data Privacy and Data Security, two espoused concepts that gain substantial traction in the current times of Covid-19.'
lastmod: '2020-06-11T20:46:05+02:00'
featured: no
image:
  caption: '[Photo by Scott Webb on Pexels](https://www.pexels.com/de-de/foto/ausrustung-burgersteig-gehweg-mauer-430208/)'
  focal_point: ''
  preview_only: true
projects: []
---

Data Privacy and Data Security, two espoused concepts that gain substantial traction in the current times of Covid-19, especially in the context of the upcoming German Corona tracking app [^1] that is about to be released next week. Public expectations compete with individual reservations and data privacy concerns, partly fueled by events such as the 2013 Snowden revelations or the 2018 Camebridge Analytica scandal. Likewise, events like these contributed to the willingness of the EU to enforce supranational regulations on data privacy and security, climaxing the in the EU-GDPR [^2] which is binding for all EU member states since 2018-05-25.

In this blog post, I would like to take the current debate as an opportunity to dive into a dataset about monetary fines since the issuance of the EU-GDPR. In the process of doing so, I will do some web scraping to extract the dataset from the *GDPR Fines Tracker* on *privacyaffairs.com* [^3], clean the dataset using some of the tools recently released as part of the `dplyr v1.0.0` release [^4], experiment with some fuzzy string matching capabilities provided in the `stringdist` package [^5] and engage in some visual EDA. Meanwhile, this post is inspired by [David Robinson](https://www.youtube.com/watch?v=EVvnnWKO_4w)'s TidyTuesday Series. The whole analysis is embeeded into an R markdown document.



### Load Packages


```r
if (!require("pacman")) install.packages("pacman")
pacman::p_load(rvest, tidyverse, kableExtra, extrafont, stringdist)
```

### Web Scraping

First, I extract the dataset from the *GDPR Fines Tracker* on *privacyaffairs.com* using the `rvest` package and some regular expression (*regex*) to finally transform the JSON-format into a convenient data frame respectively tibble format.

```r
gdpr_data <- read_html("https://www.privacyaffairs.com/gdpr-fines/") %>% 
  #find html_node that contains the data
  html_nodes(xpath = "(.//script)[2]") %>% 
  #extract text
  rvest::html_text() %>%
  #trim to produce clean json format
  str_sub(start = str_locate(., "\\[")[[1]], end = str_locate(., "\\]")[[1]]) %>% 
  #remove new lines and carriage returns
  str_remove_all(paste("\\n", "\\r", sep = "|")) %>% 
  #parse JSON
  jsonlite::fromJSON()
```

In total, 339 violations are categorized by *PrivacyAffairs* as of the day of this blogpost (2020-07-12). I randomly sample five observations to get a first overview of the dataset.


```
>    id           name  price                 authority       date
> 1   6        Romania  20000 Romanian National Supe... 10/09/2019
> 2  86 United Kingdom      0  Information Commissioner 07/10/2019
> 3 113          Spain    900 Spanish Data Protectio... 11/07/2019
> 4  87 United Kingdom  80000  Information Commissioner 07/16/2019
> 5 306         Norway 283000 Norwegian Supervisory ... 05/19/2020
>                  controller           articleViolated                      type
> 1          Vreau Credit SRL Art. 32 GDPR, Art. 33 ... Failure to implement s...
> 2 Driver and Vehicle Lic...                   Unknown Non-compliance (Data B...
> 3      TODOTECNICOS24H S.L.              Art. 13 GDPR Information obligation...
> 4   Life at Parliament View  Data Protection Act 2018 Non-compliance (Data B...
> 5       Bergen Municipality Art. 5 (1) f) GDPR, Ar... Failure to implement s...
>                      source
> 1 https://www.dataprotec...
> 2 https://www.autoexpres...
> 3 https://www.aepd.es/re...
> 4 https://ico.org.uk/abo...
> 5
```

### Data Cleaning

In a next step, I streamline the dataset a little more and try to get rid of the various inconsistencies in the data entries. 

First, I would like to adjust some of the column names to highlight the actual content of the features. I rename `name` to `country`, `controller` to `entity` (i.e. the entity fined as a result of the violation) and `articleViolated` to `violation`. Second, I infer from the sample above that the violation `date` is not in standard international date format (`jjjj-mm-dd`). Let's change this using the `lubridate` package while properly accounting for 15 `NA`s (indicated by unix time `1970-01-01`). Third, I format the `price` feature by specyfing it as a proper currency (using the `scales` package). Fourth, and analogue to the violation `date`, I properly account for `NA`s -- taking the form "Not disclosed", "Not available" and "Not known" -- in the `entity` as well as `type` feature (containing 38 and 2 missing values, respectively). In addition, I must correct one errorneous fined entity (violation id 246). Finally, I clean the `violation` predictor using regex and the `stringr` package.

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

Since a cross-check of the entity names reveals quite a few inconsistencies in how the entities have been written in the databse, I leverage the `stringdist` package for fuzzy string matching to homogenize some of the entries. For example, the *optimal string alignment* (*osa*) measure allows to assess the similarity of two strings by enumerating the number of pre-processing steps (deletion, insertion, substitution and transposition) necessary to transform one string into another. Adhering to the following assumptions yields several fuzzy matches which are accounted for in the subsequent EDA:
* Set the minimum-osa threshold to 3 (i.e. only consider string pairs which require three transformations to be aligned).
* Only consider strings of length > 3 (otherwise the minimum-osa threshold becomes redundant).

```
> # A tibble: 9 x 5
>   ent_a                          ent_b                           osa  id.a  id.b
>   <chr>                          <chr>                         <dbl> <int> <int>
> 1 Telecommunication Service Pro~ Telecommunication service pr~     2     7    48
> 2 A mayor                        Mayor                             3    32   100
> 3 A.P. EOOD                      L.E. EOOD                         2    45   204
> 4 A.P. EOOD                      T.K. EOOD                         2    45   205
> 5 A bank                         Bank                              3    50    55
> 6 A bank                         bank                              2    50   225
> 7 Bank                           bank                              1    55   225
> 8 Vodafone Espana                Vodafone España                   1    64   156
> 9 L.E. EOOD                      T.K. EOOD                         2   204   205
```


```r
gdpr_data <- gdpr_data %>% 
  mutate(across(entity,
                ~str_trim(.) %>% 
                  str_replace_all(.,
                    c("Telecommunication Service Provider" = "Telecommunication service provider",
                      "A mayor" = "Mayor",
                      "A bank" = "Bank",
                      "bank" = "Bank",
                      "Vodafone Espana" = "Vodafone España")))
                )
```

Finally, let's have a look at the cleaned data.

```
>    id        country  price                 authority       date
> 1   6        Romania  20000 Romanian National Supe... 2019-10-09
> 2  86 United Kingdom     NA  Information Commissioner 2019-07-10
> 3 113          Spain    900 Spanish Data Protectio... 2019-11-07
> 4  87 United Kingdom  80000  Information Commissioner 2019-07-16
> 5 306         Norway 283000 Norwegian Supervisory ... 2020-05-19
>                      entity                 violation                      type
> 1          Vreau Credit SRL Art. 32 GDPR, Art. 33 ... Failure to implement s...
> 2 Driver and Vehicle Lic...                   Unknown Non-compliance (Data B...
> 3      TODOTECNICOS24H S.L.              Art. 13 GDPR Information obligation...
> 4   Life at Parliament View  Data Protection Act 2018 Non-compliance (Data B...
> 5       Bergen Municipality Art. 5 (1) f) GDPR, Ar... Failure to implement s...
>                      source
> 1 https://www.dataprotec...
> 2 https://www.autoexpres...
> 3 https://www.aepd.es/re...
> 4 https://ico.org.uk/abo...
> 5
```

Now let's briefly validate the integrity of the scraped dataset.

```
> # A tibble: 1 x 11
>      id picture country price authority  date entity violation  type source
>   <int>   <int>   <int> <int>     <int> <int>  <int>     <int> <int>  <int>
> 1     0       0       0    11         0    15     38         0     2      0
> # ... with 1 more variable: summary <int>
```

And indeed, a quick glance at the missing values per feature reveals numerous missing values for the `price` (3.24%), `date` (4.42%) and `entity` (11.21%) feature. Without diving deeper into the information sources, it may be assumed that for the affected court cases no complete record of the verdict was openly published by the jurisdiction.

Also, with regards to some of the fines, *PrivacyAffairs* explicitely states that "*The Marriott and British Airways cases are not final yet and the fines are just proposals. Other GDPR fines trackers incorrectly report those as final.*"

### Exploratory Data Analysis


Finally, the cleaned data allows for some interesting exploratory data analyses. In a first step, the data reveals the *Bulgarian Commission for Personal Data Protection* as the first ever authority imposing a punishment for the violation of the GDPR on 2018-05-12. Comparing this date to the enactment of the GDPR (2018-05-25) it raises the question how a fine could have been imposed 13 days prior to the GDPR coming into effect.

```r
gdpr_data %>% 
  select(-summary, -picture) %>% 
  filter(date == min(date, na.rm = TRUE)) %>% 
  mutate(across(where(is.character), str_trunc, 25))
```

```
>   id  country price                 authority       date entity
> 1 78 Bulgaria   500 Bulgarian Commission f... 2018-05-12   Bank
>                   violation                      type                    source
> 1 Art. 5 (1) b) GDPR, Ar... Non-compliance with la... https://www.cpdp.bg/?p...
```

Having identified the first ever fine, the natural question arises: Which are the *biggest* fines ever fined?
With almost double the fee compared to the second place, Google takes the throne for the receiving the highest fine to date of 50,000,000€, imposed by the French Data Protection Authority at the beginning of 2019. Beside Google (which is by the way represented by twice in the top 10), the top 10 fined entities also include a telecom provider, a postal service provider and a real estate enterprise. Interestingly, the 10th spot is taken by public health insurance provider in Germany suggesting that the GDPR is not only directed towards the big corporations but applies equally to public bodies.

```r
gdpr_data %>%
  select(-summary, -picture) %>% 
  slice_max(order_by = price, n = 10) %>% 
  mutate(across(where(is.character), str_trunc, 25))
```

```
>     id  country    price                 authority       date
> 1   66   France 50000000 French Data Protection... 2019-01-21
> 2  200    Italy 27800000 Italian Data Protectio... 2020-02-01
> 3   79  Austria 18000000 Austrian Data Protecti... 2019-10-23
> 4   82  Germany 14500000 Data Protection Author... 2019-10-30
> 5  138  Germany  9550000 The Federal Commission... 2019-12-09
> 6  189    Italy  8500000 Italian Data Protectio... 2020-01-17
> 7  237   Sweden  7000000 Data Protection Author... 2020-03-11
> 8  190    Italy  3000000 Italian Data Protectio... 2020-01-17
> 9   15 Bulgaria  2600000 Data Protection Commis... 2019-08-28
> 10 322  Germany  1240000 Data Protection Author... 2020-06-30
>                       entity                 violation
> 1                Google Inc. Art. 13 GDPR, Art. 14 ...
> 2     TIM - Telecom Provider           Art. 58(2) GDPR
> 3              Austrian Post Art. 5 (1) a) GDPR, Ar...
> 4         Deutsche Wohnen SE Art. 5 GDPR, Art. 25 GDPR
> 5           1&1 Telecom GmbH              Art. 32 GDPR
> 6             Eni Gas e Luce Art. 5 GDPR, Art. 6 GD...
> 7                     Google Art. 5 GDPR, Art. 6 GD...
> 8             Eni Gas e Luce  Art. 5 GDPR, Art. 6 GDPR
> 9    National Revenue Agency              Art. 32 GDPR
> 10 Allgemeine Ortskranken... Art. 5 GDPR, Art. 6 GD...
>                         type                    source
> 1                    Several https://www.cnil.fr/en...
> 2  Non-cooperation with D... https://www.garantepri...
> 3  Non-compliance with la... https://wien.orf.at/st...
> 4  Failure to comply with... https://www.lexology.c...
> 5  Failure to implement s... https://www.bfdi.bund....
> 6  Non-compliance with la... https://www.gpdp.it/we...
> 7  Failure to comply with... https://www.datainspek...
> 8  Non-compliance with la... https://www.gpdp.it/we...
> 9  Failure to implement s... https://www.cpdp.bg/in...
> 10 Failure to implement s...
```

Altogether, the 10 million € has only been cracked by five individual entities so far. Given that [Art. 83 GDPR No. 4](https://gdpr.eu/article-83-conditions-for-imposing-administrative-fines/) explicitely uses the 10 million € threshold (or alternatively 2% of total revenues) as whip to emphasize the potential consequences of a violation, it appears that most firms have not yet stressed this limit -- and correspondingly also most firms have not yet stressed the even more extreme limit of 20 million € specified in [Art. 83 GDPR No. 5](https://gdpr.eu/article-83-conditions-for-imposing-administrative-fines/).

```r
gdpr_data %>%
  drop_na(price) %>% 
  mutate_at(vars(entity), ~as.factor(.) %>% 
              fct_lump_n(., 8, w = price) %>% 
              fct_explicit_na(., na_level = "Other")) %>% 
  group_by(entity) %>% 
  summarise(total_fine = sum(price),
            fine_freq = n(),
            .groups = "drop") %>% 
  
  
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
```

<img src="index_files/figure-html/plot1-1.png" width="70%" style="display: block; margin: auto;" />

Extending the EDA to the whole landscape of fines illustrates that the very large fines (colourized in the figure) are indeed rather rare. The majority of fines populate the area below the 100,000€ threshold with the top 10 defending the upper parts of the graph. In addition, the figure yields some more information about the distribution of fees across time. That is, penalties are imposed reluctantly in 2018 and more frequently starting with the year 2019. One may argue, that the authorities may have viewed 2018 as a transitional period in which they may have been busy with establishing own regulatory practices and processes.
*(Note that fees are plotted on the log-scale)*

```r
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

<img src="index_files/figure-html/plot2-1.png" width="70%" style="display: block; margin: auto;" />

Looking at the countries which have imposed the biggest fines on aggregate, it is remarkable for the [German 'Aluhut'](https://de.wikipedia.org/wiki/Aluhut) to renounce the lead and hand the gold and silver medal to France and Italy.

<img src="https://tenor.com/view/alex-jones-gif-9848358" width="70%" style="display: block; margin: auto;" />


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
      subtitle = "The German 'Aluhut' Lags Behind",
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

<img src="index_files/figure-html/plot3-1.png" width="70%" style="display: block; margin: auto;" />

Further, I am curious which articles were violated most frequently. Therefore, I look at the subset of fines for which the violated GDPR article is available in the data. Since several fines relate to two or more articles at a time, I split the corresponding variable `violation` based on delimiter using `separate_rows()`.
Eventually, we Hungary and Romania belong to the countries with the most diverse set of violated articles. In contrast, for countries like Iceland or Estonia there is barely any activity on the GDPR prosecution market. Finally, it becomes evident from the plot that article 5, 6 and 32 are obviously causing the biggest problems for companies as the data records a case relating to those articles for almost any country present in the dataset.

```r
gdpr_data %>% 
  select(country, violation) %>% 
  separate_rows(violation, sep = ",") %>% 
  filter(str_detect(violation, "Art.")) %>% 
  mutate(across(violation, ~str_extract(., "(?<=Art.\\s?)[0-9]+"))) %>%
  mutate(across(everything(), as.factor)) %>% 
  group_by(country, violation) %>% 
  summarise(freq = n(), .groups = "rowwise") %>% 
  
  ggplot(aes(
    x = fct_relevel(violation, function(x){as.character(sort(as.integer(x)))}),
    y = country
  )) +
    geom_point(aes(color = freq), shape = 15, size = 2) +
    scale_color_gradient(name = "Number\nof Fines", low = "#cdd8ff", high = "#021868") +
    labs(
      x = "GDPR Article Number",
      y = "",
      title = "Distribution of Violated Articles across Europe",
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

<img src="index_files/figure-html/plot4-1.png" width="70%" style="display: block; margin: auto;" />

Shifting the view a little, and asking the question which article incurred the highest average fine, we again find article 5, 6 and 32 on the front spots. For this plot, I joined the data with the respective article titles to give more meaning to the numbers themselves. Moreover, I assumed that a fine relates proportionally to all articles mentioned in the respective case by allocating the same share to each article involved in the fine.
Strangely, one or more violations of [Art. 58 GDPR](https://gdpr.eu/article-58-supervisory-authority-investigative-powers/), titled 'Powers', supposedly lead to substantial penalties -- strange in the sense that the contents of the article rather specifies the investigative powers of the supervisory authority, rather than explicitely regulating the data-related practices of the economic entities...

```r
gdpr_data %>% 
  drop_na(price) %>% 
  select(violation, price) %>% 
  mutate(across(price, ~ . / str_count(violation, "Art."))) %>% 
  separate_rows(violation, sep = ",") %>% 
  filter(str_detect(violation, "Art.")) %>% 
  mutate(across(violation, ~str_extract(., "(?<=Art.\\s?)[0-9]+") %>% as.factor %>% fct_lump(., 8, w = price))) %>%
  group_by(violation) %>% 
  summarise(avg_fine = sum(price),
            .groups = "rowwise") %>% 
  ungroup %>% 
  left_join(read_delim("./_article_titles.txt", delim = "; ", col_types = cols("f", "c")), by = c("violation" = "article")) %>% 
  mutate(across(title, ~paste(violation, "-", .))) %>% 
  
  
  ggplot(aes(avg_fine / 1000000,
             title %>% str_trunc(50) %>% str_wrap(30) %>% as.factor %>% fct_reorder(., avg_fine, .desc = F))) +
    geom_col(fill = signature_color) +
    scale_x_continuous(labels = euro) +
    labs(
      x = "Imposed Fine [m€]",
      y = "GDPR Article",
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

<img src="index_files/figure-html/plot5-1.png" width="70%" style="display: block; margin: auto;" />

Finally, I am also curious about the distribution of penalties throughout the year. Using the `coord_polar()` function to transform the `geom_col` mapping into a circular representation. From this approach to visualizing the distribution it becomes evident that Februray and June appear to form the so-callded *busy season*. On the contrary, the plot may lead to suggest that the July-September period represents the general vacation period: Either the firms are less eager in violating GDPR regulations or the authorities are less active in pursuing potential violations.

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
      subtitle = "Hello, Summer - We're on Vacation",
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

<img src="index_files/figure-html/plot6-1.png" width="70%" style="display: block; margin: auto;" />

Either way, the upcoming month in the GDPR prosecution domain promise to be rather calm -- one reason more for me to finally take a (hopefully) well-deserved vacation...

### References

[^1]: SAP; Deutsche Telekom (2020): Corona-Warn-App: The official COVID-19 exposure notification app for Germany, Github 2020, URL: https://github.com/corona-warn-app (accessed: 2020-06-11).
[^2]: European Union (2018): General Data Protection Regulation (GDPR), European Union 2018, URL: https://gdpr.eu/tag/gdpr/ (accessed: 2020-06-11).
[^3]: PrivacyAffairs (2020): GDPR Fines Tracker & Statistics, PrivacyAffairs 2020, URL: https://www.privacyaffairs.com/gdpr-fines/ (accessed: 2020-06-11).
[^4]: Wickham, H. (2020): dplyr 1.0.0 available now!, Tidyverse 2020, URL: https://www.tidyverse.org/blog/2020/06/dplyr-1-0-0/ (accessed: 2020-06-11).
[^5]: van der Loo, M. P.J. (2014): The stringdist Package for Approximate String Matching, in: The R Journal, Vol. 6, No. 1, 2014, pp. 111‑122.
