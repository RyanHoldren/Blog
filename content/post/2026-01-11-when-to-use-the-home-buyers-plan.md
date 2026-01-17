---
title: "When to Use The Home Buyers’ Plan"
date: 2026-01-15
author: "Ryan Holdren"
---

## Disclaimer

I'm not a tax professional, accountant, or financial advisor. This blog post is for informational and educational purposes only. It's not intended to be a substitute for personalized and professional tax advice.

## The Home Buyers’ Plan

I've been thinking recently about the **Home Buyers’ Plan** (HBP), which is a quirky Canadian financial rule where you can borrow up to $60,000 from your Registered Retirement Savings Plan (RRSP) and then repay it over the next (at most) 17 years, with no added interest, designed to help you buy a home.

I have been trying to think through the economics of this rule. If you don't actually _need_ to use the Home Buyers’ Plan, is it a good financial move to use it anyway?

## Reframing The RRSP

To answer this question, let's first reframe _how_ we think about the RRSP. Instead of thinking about it as _one_ bucket of money, let's split it up by **ownership**:

- There is **your share** of your RRSP: the portion that ultimately ends up in your pocket. The growth of this share is **tax‑free** while it is inside the RRSP much like a Tax‑Free Savings Account (TFSA). Also like a TFSA, the withdrawal of this share is tax‑free.

- Then there is the **government's share** of your RRSP: the portion that ultimately ends up being remitted as income tax, at time of withdrawal. The growth of this share accrues entirely to the government.

```
Let `X` = your marginal tax rate at the time of withdrawal.
```

How much of your RRSP is your share and how much of it is the government's? We can model it, however; we can say with certainty that your share is `1 - X` and the government's is the remaining `X`.

For example, if your marginal tax rate when you withdraw will be 25%, then one-quarter of the _present_ value of your RRSP effectively belongs to the government, and any growth on that quarter likewise accrues to the government. The remaining three-quarters are yours, and will grow tax-free inside the plan, like a TFSA.

## Reframing The Home Buyers’ Plan

Keeping that in mind, let's reframe how the HBP actually works, when you borrow a dollar:

1) Some of this dollar is borrowed from the government's share.
2) The rest of the dollar is borrowed from your share.
3) You invest the entire dollar outside the RRSP, where returns are taxable, not tax‑free.
4) You get to keep the nominal, after-tax growth of the entire dollar, while it's outside your RRSP.
5) Eventually, you repay the dollar back into your RRSP, restoring the balance.

Reframed in this way, it's clear that what you have gained is that you got to borrow the government’s share at 0% interest, and keep the **taxable** growth of that share while it's outside your RRSP. What you have sacrificed, on the other hand, is the tax-free growth on *your share* of the dollar that you borrowed from your RRSP; you still get the growth, but it won't be tax-free anymore.

## Analysis

```
Let `rₚ` = your _current_ **pre‑tax** rate of return **inside** your RRSP.
Let `rₘ` = your _current_ **after‑tax** rate of return **outside** your RRSP.
```

**Note** that if you are using the HBP to make a larger down payment, then `rₘ` is just the interest rate on your mortgage. Your current marginal tax rate is irrelevant in this case, because debt repayment is effectively a tax-free investment.

**Note** that we are assuming that `X` does not change, depending on whether or not you use the HBP. This is almost certainly false; having a smaller RRSP and more assets outside of your RRSP is likely to _impact_ your marginal tax rate. It's really hard to say more than that though, since we don't know what the tax system will look like, decades from now.

Let’s compare two options:

1) You could borrow one dollar from your RRSP via the HBP and invest it outside your RRSP for _exactly_ one year, then you pay back the dollar. The benefit of this option is that the dollar will grow at `rₘ`. At the end of the year, you will have gained `$1 × rₘ`. 

2) Alternatively, you could leave the dollar in your RRSP. The benefit of this option is that _your_ share of that dollar will grow at `rₚ`. At the end of the year, you will have gained `$1 × (1 - X)rₚ`.

You will notice that I've reduced the problem to a single period. This simplifies the problem greatly, and I think it's _technically_ valid, since you can repay the HBP in full at any time, so you really do have the option to unwind if `rₚ` or `rₘ` change.

We can therefore say that the HBP is beneficial if:

```
rₘ > (1 - X)(rₚ)
rₘ > rₚ - X * rₚ
rₘ - rₚ > 0 - X * rₚ
rₚ - rₘ < X * rₚ
(rₚ - rₘ) / rₚ < X
X > (rₚ - rₘ) / rₚ
```

I put together [a table of the break-even value of `X`](https://docs.google.com/spreadsheets/d/1Ck4-MDGYVKNFdqdOIjWDF7aFTzN6NDA6kBsNHBxMl04/edit?usp=sharing) for different values of `rₚ` and `rₘ`.

For example, at the time of writing, the financial planning assumptions of PWL Capital suggest that an all-equity portfolio can be expected to return around 7% before-tax, net-of-fees. Meanwhile, fixed five-year mortgage rates are around 4%. Substituting those values for `rₚ` and `rₘ` respectively, you can see that the break-even value of `X` is about 43%. This means that if your marginal tax rate at the time of withdrawal will be _above_ 43%, then you _should_ use the HBP to make a larger down payment. On the other hand, if your marginal tax rate at the time of withdrawal will be _below_ 43%, you'd be better off leaving the money in your RRSP.

## Leverage

One observation worth noting and considering is how, depending on what you do with your HBP loan, you are effectively leveraging or deleveraging. If you borrow via the HBP to increase your down payment by a dollar, your net debt has decreased by `$1 × (1 - X)`. On the other hand, if you invest the dollar in a non-registered account, your net debt has increased by `$1 × X`.

If you are not going to use the HBP to increase the size of your downpayment, but instead are going to invest it in a non-registered account, you are using leverage. This should be taken into consideration, since leverage will magnify your risk.