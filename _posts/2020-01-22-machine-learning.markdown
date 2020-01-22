---
layout: post
title:  "Effective (and Simple!) Predictive Recommendations"
excerpt: ""
date:   2020-01-22 12:36:00
categories: 
tags: machine-learning recommendations
---

I want to give a quick illustration of the idea that effective applications of machine learning -- especially early versions -- can be almost embarrassingly simple. This is the broad strokes of a system I implemented that produced a substantial lift in revenue for a large online marketplace. 

This marketplace is a bit unusual in that its products are all one of a kind. Every day, we’d list a new batch of products for sale, and the problem I faced was how do I pick which products to recommend to each user. 

That each product was unique meant that the most common solutions for generating recommendations like a correlation matrix couldn’t be applied since we don’t have historical data about what users who’ve bought the same products had also bought. I did notice, however, when I looked into our data that users had a high affinity for particular types of products. At a coarse level, that meant we could use our product categorization as the basis for recommendations.

The implementation is simple and took less than a plane ride to finish. I wrote a background job that scored each user for their affinity for a category based on usage signals we had available (say, viewing a product, saving a product a wishlist (better signal than just viewing!), purchasing a product (great signal!)), producing a set of scores normalized across our categories that represented each user’s preferences. Those scores were then used when a known user viewed new products to predict which they’d be most interested in.

Split tests showed a substantial lift in revenue from its introduction.

Notes:
* There are, of course, a few wrinkles when deploying a system like this. One is the idea that our confidence in our predictions should be proportional to how well we know that user. The less that’s known about a user, the easier it is to make predictions that overfit for the little we do know.
* Predictive recommendations for engagement are important, but definitely don’t speak to the whole picture for either a user or the business. The ranking system I wrote also incorporated factors like anticipated customer reviews, support workload, customer churn, profitability, etc.

