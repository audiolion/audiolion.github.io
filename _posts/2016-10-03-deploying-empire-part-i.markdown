---
layout: post
title:  "Deploying Empyre - Part I"
date:   2016-10-03 15:05:15
author: Ryan Castner
categories: Empyre
tags: empyre django postgres psql
cover:  "/assets/empire.jpg"
---

## What is Empyre

[Empyre][empyre] is my first forray into the world of quantitative finance. Quantitative finance is the application of mathematical models to financial data to analyze, predict and model the financial world. My motivation for the project is mainly academic, with the goal being to apply a range of skills I have learned throughout my schooling to create a useful web application. The secondary motivation is of course that it might one day be useful when I have money to invest. Empyre seeks to be a web app that provides traders indicators on markets to help them trade stocks.

I will not go any further into the project for now, and will later be doing a full write-up on the project specification which I will link back to. For now let us dive into the purpose of this series, which will be to show I set up a Virtual Private Server and created the initial scaffolding and setup for a Django web application. The series will go heavily into systems administration, one of the areas of expertise required for full stack developers. I will discuss motivations for my choices and try to explain the reasons behind decisions without going to deep down the rabbit hole.

## Provisioning a Server


[empyre]: https://github.com/audiolion/empyre.git