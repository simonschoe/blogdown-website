---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: >
  The Transfer Market Madness: Determinants and Predictions of Football Player Transfer Fees
subtitle: "*A machine learning project concerned with estimating the future transfer fees of European football players realized in R.*"
authors: ["simon"]
tags: [Football Analytics, Transfer Fees, Player Popularity, Predictive Modelling, Regression]
categories: []
date: 2019-12-20


# Summary. An optional shortened abstract.
summary: A machine learning project concerned with estimating the future transfer fees of European football players realized in R.

# Optional external URL for project (replaces project detail page).
external_link: ""

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  placement: 1
  caption: "© [Markus Spiske](https://www.pexels.com/de-de/foto/himmel-sonnenuntergang-feld-sonnenaufgang-114296/)"
  focal_point: "TopLeft"
  preview_only: true

# Custom links (optional).
#   Uncomment and edit lines below to show custom links.
# links:
# - name: Follow
#   url: https://twitter.com
#   icon_pack: fab
#   icon: twitter

url_code: "https://github.com/simonschoe/project_player_transfers"
url_pdf: "football_player_transfers.pdf"
url_slides: ""
url_video: ""

# Slides (optional).
#   Associate this project with Markdown slides.
#   Simply enter your slide deck's filename without extension.
#   E.g. `slides = "example-slides"` references `content/slides/example-slides.md`.
#   Otherwise, set `slides = ""`.
slides: ""

share: false
commentable: true 
---

In the recent years, excessive transfer fees for football players have turned the European transfer market upside down – a development coined as the ‘transfer market madness’ in this paper. Predicting those fees is assumed to be of great interest to researchers, policy makers, managers and the public alike. This paper addresses the issue by discussing the following four research questions: (1) What are the important value drivers that determine transfer fees in the European football market? (2) Is there a significant influence of a player’s popularity on transfer fees? (3) Which model for estimating football transfer fees performs best in terms of predictive accuracy? (4) Where is the transfer market madness heading to?

For this purpose, five predictive modelling techniques from the regression family are proposed: linear, stepwise forward, Ridge, Lasso and polynomial regression. The models are trained on a rich data set of 2,634 transfers observed during the 2013-2019 period which is scraped from transfermarkt.de, kaggle.com and Wikipedia. The empirical results reveal numerous important predictors for the transfer fee and especially indicate that the median player transfer incorporates a €1,06 million price premium that accounts for a player’s popularity. Moreover, the quadratic regression model yields the overall best predictive accuracy with the 1-standard error-rule Lasso model being the least prone to overfitting. Finally, the latter is deployed and evaluated on four currently rumoured transfers. The generated predictions are not only remarkably close to what is hypothesised by the media but also suggest that the transfer market madness is there to stay.

Eventually, the paper advocates the use of novel predictors and consideration of non-linear relationships in future research. From a practical perspective, this study develops a tool that can aid managers and agents in terms of decision-making. In addition, concerns regarding the ethical and psychological implications of the transfer market development are raised leading to the question: When will the excessive costs become unbearable for the average team as well as the players and how will it shape the competitive equilibrium in the future of football?
