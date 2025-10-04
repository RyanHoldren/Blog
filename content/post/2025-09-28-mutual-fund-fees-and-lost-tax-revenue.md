+++
date = '2025-10-04T9:00:00-07:00'
title = 'Mutual Fund Fees and Lost Tax Revenue'
+++

In Canada, there are several tax-deferred accounts for retirement investing:

1. RRSP (Registered Retirement Savings Plan)
2. RRIF (Registered Retirement Income Fund)
3. LIRA (Locked-In Retirement Account)
4. LIF (Life Income Funds)

These accounts all have different rules, but they share one important feature: withdrawals are taxed as income. This is the meaning of "deferred" in "tax-deferred" investing. The tax isn't avoided; it's just postponed. In fact, the larger the account balance grows, the larger the eventual tax bill.

With that in mind, let's pull some numbers:

1. According to [StatCan’s Survey of Financial Security](https://www150.statcan.gc.ca/t1/tbl1/en/tv.action?pid=1110001601), the combined value of tax-deferred retirement accounts was about **$1.94 trillion** in 2023.

2. According to [the 2024 Investment Funds Report](https://www.sima-amvi.ca/wp-content/themes/ific-new/util/downloads_new.php?id=30189&lang=en_CA) by the Investment Funds Institute of Canada, **about 80% of this is held in mutual funds**.

3. According to another [frequently cited](https://www.advisor.ca/industry-news/fund-industry-generated-nearly-31b-from-mers-in-2023/) report by the Investment Funds Institute of Canada, the average **management expense ratio of these mutual funds was 1.47%** in 2023. If this seems high, it is! In their [Global Investor Experience Study](https://www.morningstar.com/content/cs-assets/v3/assets/blt9415ea4cc4157833/bltb8ac1ee320183746/66164d249cd155ef73550553/GIE_2022.pdf) on Fees and Expenses in 2022, Morningstar gave Canada a "Below Average" grade; only Italy and Taiwan did worse. For contrast, the average expense ratio for American mutual funds is just 0.47%, [according to Morningstar](https://www.morningstar.com/articles/1137689/the-average-etf-expense-ratio-is-dropping).

4. Even in Canada, there are much cheaper alternatives. For example, there are low-cost Exchange Traded Funds that make it straightforward to build a diversified portfolio with a management expense ratio below **0.20%** such as [XBAL](https://www.blackrock.com/ca/investors/en/products/239449/ishares-balanced-income-coreportfoliotm-fund) or [ZBAL](https://www.bmogam.com/ca-en/products/exchange-traded-fund/bmo-balanced-etf-zbal#overview).

5. There is no publicly available data on the effective marginal income tax rate of seniors withdrawing from these accounts, so we'll have to do some estimating. In [a previous post](https://holdren.ca/post/2025-09-20-staggering-taxable-income-for-high-income-seniors/), I illustrated how the effective marginal income tax rates for seniors in Canada are surprisingly high. Looking at those rates, I think it's unlikely that the marginal effective tax rate on withdrawals is less than 20%; retirees with income that low probably don't have any retirement savings at all. I'll estimate that **30%** is the average; some retirees will pay a little less, others considerably more.

Now, let's explicitly state a few assumptions:

1. The direct effect of fees is to reduce retirees' returns. While it's possible for an individual active investor to beat the market sufficiently to cover some or all of their fees, this isn't possible **in aggregate** because active investing is a zero-sum game; any active investor that beats the market **must** do so at the expense of other active investors. That said, not every active investor is a mutual fund manager. Perhaps, as a category, they beat the market (gross of fees) at the expense of another category (e.g., day traders)? It could also be the other way around, with hedge funds *eating their lunch*. I tried to find evidence of this one way or the other and failed, so I'm defaulting to the assumption that mutual fund managers, in aggregate, do not beat the market, gross of fees.

2. The Canadian financial sector is filled with intelligent and capable professionals, and there is no shortage of useful work to be done in Canada. Instead of quietly draining retirement savings, they could find other productive uses for their talents. 

Given all of the above, we can do some math:

```
Lost Government Revenue = Total Assets × Portion in Mutual Funds × Excess Fees × Average Tax Rate
                        ≈ $1,939,350,000,000 × 80% × (1.47% - 0.20%) × 30%
                        ≈ $5,911,138,800
```

On this basis, high fees in tax-deferred accounts are costing the government almost **$6 billion in tax revenue every year**.