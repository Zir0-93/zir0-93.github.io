--- 
title:  "Deploy A Production-Ready E-Commerce Solution on AWS with CloudFormation"
date:  2020-09-11 15:04:23
og_image: https://raw.githubusercontent.com/hadii-tech/production-ready-prestashop/master/resources/scalable_presta.png
redirect_to: "https://github.com/hadi-technology/production-ready-prestashop"
tags: [aws, ecommerce, cloudformation, scalability]
description: "Vendor lock-in and transaction fees on hosted e-commerce platforms compound at scale. This post walks through a CloudFormation reference architecture for PrestaShop on AWS: an auto-scaling application tier, managed RDS, ElastiCache and a CDN, all defined as code and reproducible with one command."
excerpt_separator: <!--more-->
---

E-commerce platforms like Shopify have dominated the e-commerce industry for many years due to their low upfront setup costs and ease of use.
However, after a recent engagement with a large online merchant, our team found two significant reasons as to why a commercial solution such as Shopify may not always make sense for an organisation:

<div style="border:1px solid rgba(15,23,42,0.08);border-radius:12px;padding:14px 18px;margin:16px 0;background:rgba(255,255,255,0.6);">
<p style="margin:0 0 8px 0;font-size:0.75rem;font-weight:700;letter-spacing:0.08em;text-transform:uppercase;color:#6b7280;">Relevant Repos</p>
<div style="display:flex;flex-wrap:wrap;gap:6px;align-items:center;">
<a href="https://github.com/hadii-tech/production-ready-prestashop"><img src="https://img.shields.io/badge/AWS-Prestashop-ff69b4?logo=github" alt="production-ready-prestashop"></a>
</div>
</div>
<!--more-->
