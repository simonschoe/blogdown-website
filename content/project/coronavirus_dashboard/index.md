---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: >
  Coronavirus (SARS-CoV-2) Dashboard based on JHU CSSE data
subtitle: "*Data Visualization Project*"
authors: ["simon"]
tags: [Data Visualization, Coronavirus, Dashboard, Tableau]
categories: []
date: 2020-04-18


# Summary. An optional shortened abstract.
summary: A data visualization project concerned with the spread of the novel Coronavirus realized in R and Tableau.

# Optional external URL for project (replaces project detail page).
external_link: ""

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  placement: 1
  caption: "© [Markus Spiske](https://www.pexels.com/de-de/foto/nummern-notfall-alarm-warnung-3970330/)"
  focal_point: "TopLeft"
  preview_only: true

# Custom links (optional).
#   Uncomment and edit lines below to show custom links.
# links:
# - name: Follow
#   url: https://twitter.com
#   icon_pack: fab
#   icon: twitter

url_code: ""
url_pdf: ""
url_slides: ""
url_video: ""

# Slides (optional).
#   Associate this project with Markdown slides.
#   Simply enter your slide deck's filename without extension.
#   E.g. `slides = "example-slides"` references `content/slides/example-slides.md`.
#   Otherwise, set `slides = ""`.
slides: ""

share: true
commentable: true 
---
This is my version of a Coronavirus dashboard, designed entirely using the data visualization tool Tableau. The dark theme and layout is obviously inspired by the well-know dashboards provided by the [Robert Koch-Institut](https://experience.arcgis.com/experience/478220a4c454480e823b17327b2bf1d4) respectively the [Johns Hopkins Coronavirus Resource Center](https://coronavirus.jhu.edu/map.html).

The data dataset is drawn from three different sources:
- SARS-CoV-2 data provided by the [JHU CSSE](https://github.com/CSSEGISandData/COVID-19),
- Population data provided by [The World Bank](https://data.worldbank.org/indicator/sp.pop.totl) for the year 2018,
- Governmental intervention data provided by [ACAPS](https://www.acaps.org/covid19-government-measures-dataset). Thanks to [Joachim Gassen](https://joachim-gassen.github.io/2020/03/merge-covid-19-data-with-governmental-interventions-data/) for this additional idea.

The data in `R` and cleaned using the tools provided in `tidyverse` R-package collection. Country names are standardized using the regex-parser implemented in the `passport` R-package. In addition to the features contained in the merged dataset, the following features have been generated:
- `Total Closed Cases`, `Total Active Cases`,
- `Incremental Confirmed Cases`, `Incremental Recovered Cases`, `Incremental Dead Cases`, `Incremental Closed Cases`, `Incremental Active Cases`,
- the `log` of each all total and incremental termed features,
- the `Infection-Recovery-Ratio` computed `Total Confirmed Cases` over `Total Recovered Cases`,
- the `Recovery-Death-Ratio` computed `Total Recovered Cases` over `Total Dead Cases`,
- the `Active-Closed-Ratio` computed `Total Active Cases` over `Total Closed Cases`,
- the version of the case fatality rate (CFR) computed as the `Total Dead Cases` relative to the `Total Confirmed Cases` (*CFR1*), the sum of `Total Recovered Cases` and `Total Dead Cases` (*CFR2*) respectively `Total Confirmed Cases` at `t-8` (*CFR3*) with eight assumed to be the average incubation period and
- the `Doubling Time` of the `Total Confirmed Cases` in days.


**Last data update:** 18-04-2020

*Click the image to interact*
[![SARS-CoV-2 Dashboard](preview.png)](https://public.tableau.com/views/SARS-CoV-2Dashboard_15872319731490/Global?:display_count=y&:origin=viz_share_linkfeatured.png)
