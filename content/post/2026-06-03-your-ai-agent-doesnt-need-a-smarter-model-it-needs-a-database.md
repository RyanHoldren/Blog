---
title: "Your AI Agent Doesn't Need a Smarter Model. It Needs a Database."
date: 2026-06-03
author: "Ryan Holdren"
---

**A note before we start:** if you give an AI agent access to personal data, you should make sure that data can't be exfiltrated by a malicious actor via prompt injection, network access, or some other vector.

I've been running an experiment: how much value can I get out of a personal AI assistant that uses cheap, open-source language models, on a budget of $5 per month in inference costs?

The $5 limit is not arbitrary. Frontier models are genuinely impressive, but they cost _real money_ when used as agents. Open-source models are an order of magnitude cheaper. I've tried to close some of the gap with good tool design—giving cheaper models the right interfaces so they don't need to be as smart.

I have found that even cheap models can correctly answer questions like "How has my sleep score trended since I started doing HIIT every week?" or "Where are we spending more on groceries, year-over-year?" when given a well-architected database and smart tooling.

## Why a database, not a file?

Token efficiency is central to this. Every token you put in context costs money, so every token the model wastes sifting through rows it doesn't need is money gone. The design of every tool I give my AI agent is therefore shaped by one question: what is the minimum context the model needs to answer this question correctly?

The problem with keeping personal data in files is that cheap models are notoriously bad at using them. A year of daily health readings might be tens of thousands of rows. If you ask "How was my sleep this month?" a frontier model might write a regex and `grep` the file (or use `jq`, if it's JSON, etc), but a cheap model is very likely to just try to read the whole file.

A database, on the other hand, can force an AI agent to query for exactly what it needs. Aggregations reduce data volume dramatically. A `SELECT AVG(score) FROM sleep WHERE date >= '2026-05-01'` returns *one* number instead of hundreds of rows.

There's also a freshness argument; files go stale! A database with sync tools running on cron schedules means the agent always has current data, without any effort on your part.

## Why not just give the agent API access?

The obvious alternative to a static file is to give the agent credentials for each data source and let it call the APIs directly. There are a few reasons why I didn't do this.

The biggest reason is cross-source queries. "Do I sleep better on days that I do a Peloton workout?" In a database, this question can be answered by a query that joins both tables (i.e. the sleep table and the workouts table), but it's not so simple when the data sources are completely isolated. A very smart model might, for example, fetch the data and build an ad hoc spreadsheet, but cheap models aren't smart enough for that. They will usually just scan both data sets and make a (usually inaccurate) statement about what they see.

The second is normalization. Every API returns data in its own format (i.e. different timestamp conventions, different units, inconsistent nulls, field names, etc). One API might return dates as Unix timestamps and another may use ISO 8601 strings with timezone offsets. A cron job that syncs data from an API into a database can normalize all of this, so the agent never has to deal with any of it.

Moreover, by logging all the SQL queries that your agent executes—especially the ones that fail—you have the opportunity to rename tables and columns or add better comments.

Finally, there's also a data sovereignty story to be told here. The act of normalizing all your health and finance data into a database that **you own** makes you less vendor locked. If I decide to replace my Fitbit with a Whoop, I am *still* going to be able to see how my stats have changed, year-over-year.

## Why PostgreSQL?

If you only care about LLM performance, [SQL-GEN](https://arxiv.org/abs/2408.12733) suggests SQLite has a slight edge out of the box. The authors managed to close the gap with PostgreSQL to roughly 1-3%, and since the paper is from 2024, those techniques have likely been incorporated into recent open-source models already. In practice, the dialect gap is probably negligible with a modern model. That said, there's still good reason to think PostgreSQL sits near the top: it's one of the most widely deployed databases in the world, meaning training data coverage is exceptional, and it hews closely to ANSI SQL, so anything a model has learned from the standard transfers directly.

The practical reasons to use PostgreSQL are more compelling still. Multiple cron jobs need to write to the database concurrently, which SQLite can't handle reliably. And I wanted to be able to write queries using advanced features like lateral joins and window functions, which SQLite either doesn't support or supports poorly.

PostgreSQL handles all of that. In my experience, AI models write **very good** PostgreSQL queries. Even cheap models can one-shot a complex query with CTEs, window functions, lateral joins, etc.

## Listing schemas and tables

My database interface is a small Rust binary exposed to the agent as two tools: `database_list` and `database_query`.

The first tool, `database_list`, is where everything starts. Before the agent can query a table, it needs to know the table exists. Without a schema discovery step, it'll hallucinate table names from prior context.

The tool description for `database_list` tells the agent: *"Always call this first before running a query so you know the correct table names."* This is load-bearing.

The output looks like this:

```
Here is the list of schemas and tables in your PostgreSQL database:

  fitness — Daily fitness and health metrics
    fitness.sleep (id, date, score, total_sleep, rem, deep, efficiency)
    fitness.workouts (id, date, type, duration_min, calories, heart_rate_avg)

  finance — Bank and credit card transactions
    finance.accounts (id, institution, name, type, last_synced)
    finance.transactions (id, account_id, date, amount, name, category, pending)

Use the `database_query` tool to execute a SQL query that uses these tables.
```

Each schema and table has a `COMMENT ON SCHEMA` or `COMMENT ON TABLE` set in the database. The `database_list` tool uses PostgreSQL's `obj_description()` to retrieve them. When a comment exists, it prints alongside the name.

This means the agent sees `finance — Bank and credit card transactions` instead of just `finance`. Every sync tool that creates a schema sets these comments at creation time, so there's no separate documentation to keep synchronized.

## Rendering tables in Markdown

Query results come back as Markdown tables with padded columns:

```
| date       | category    | amount  |
| ---------- | ----------- | ------- |
| 2026-05-01 | Groceries   | 127.43  |
| 2026-05-03 | Dining Out  | 62.10   |
| 2026-05-08 | Groceries   | 94.17   |

(3 rows)
```

Language models are trained on a lot of Markdown. They parse Markdown tables naturally, because the separator row makes column boundaries unambiguous. [Table Meets LLM](https://arxiv.org/abs/2305.13062) (WSDM 2024) found that input format substantially affects LLM table understanding performance. [Talking with Tables](https://arxiv.org/abs/2412.17189) found a 40% average performance gain from tabular format over JSON and other representations, with better token efficiency.

In a frontier model, these things probably wouldn't matter much, but when using cheap models, every little bit counts.

## Limiting rows

The `database_query` tool caps output at 100 rows. When the result exceeds this, the agent sees:

> Showing 100 of 847 rows. Add a WHERE clause to narrow the results or a GROUP BY to aggregate them.

There's solid research behind both the cap and the nudge toward aggregation.

* [Lost in the Middle](https://arxiv.org/abs/2307.03172) (TACL) showed that LLM performance is highest when relevant information appears at the beginning or end of the context, and degrades significantly when the model has to find it in the middle of a long input. A table with hundreds of rows buries the relevant data exactly where the model is worst at using it.

* [Context Length Alone Hurts LLM Performance Despite Perfect Retrieval](https://arxiv.org/abs/2510.05381) went further: even when a model can perfectly retrieve the relevant information, context length alone degrades performance by 14–85%. More raw data in context doesn't help; it hurts.

In practice, the agent should never scan raw rows to answer an analytical question. A `SUM()` or `AVG()` returns one number; the agent reasons over that number perfectly. Scanning 847 transaction rows to mentally total them is exactly the kind of task where a cheap model will make errors. The row cap and the tool description work together to steer it toward the right approach.

## Formatting dates and numbers

How you format values in query results matters, so the `database_query` tool handles this automatically so the agent doesn't have to think about it.

* With respect to **dates**, [Date Fragments](https://arxiv.org/abs/2505.16088) found that BPE tokenizers fragment run-together date strings into meaningless pieces—`20250312` becomes `202`, `503`, `12`—which forces the model to reconstruct the date from fragments and correlates with accuracy drops of up to 10 percentage points on uncommon dates. Hyphen-separated ISO 8601 (`2026-05-31`) avoids this by tokenizing cleanly at the separators. PostgreSQL outputs `date` columns in ISO 8601 by default, so the tool gets this right without any special handling.

* When it comes to **numbers**, [The Effect of Scripts and Formats on LLM Numeracy](https://arxiv.org/abs/2601.15251) found that LLM accuracy on numerical tasks drops substantially when numbers are formatted in ways that differ from what the model saw in training—even when the underlying value is identical. [NumeroLogic](https://arxiv.org/abs/2404.00459) found that models struggle with place value because they can't tell whether a digit represents thousands or hundreds until the entire number is read. Comma separators solve this directly—`1,234,567` signals its scale immediately, where `1234567` does not. The tool formats numbers with comma separators and currency symbols where appropriate, so `1234567` becomes `$1,234,567.00` before it ever reaches the agent.

## Handling time zones

This is subtle but very important: cheap models _struggle_ with timezones. For that reason, the `database_query` tool **requires** a `timezone` argument (e.g. "America/Vancouver"), and then, under the hood, the tool prepends a `SET TIME ZONE` statement to every query:

```sql
SET TIME ZONE 'America/Vancouver';
```

This is absolutely critical! A workout logged at 7:00 PM PDT appears as 02:00 the next day in UTC. A human reviewer is _likely_ to notice that and dig deeper—most folks aren't working out at 2:00 in the morning!—but in my observations, cheap models won't even question it.

By making the timezone an explicit, required argument, we force the agent to think about what to pass, and, probably more importantly, it will reappear in the context when the model is considering the query results.

My USERS.md explicitly states that I live in Vancouver, but I suspect that if I travelled a lot, I could convince the agent to derive the timezone from my location.

## Complexity warnings

Cheap open-source models are generally reliable on straightforward SQL. They start making mistakes on complex queries, particularly, it seems, anything involving window functions or multiple CTEs.

To combat this, before the `database_query` tool returns results, it parses the SQL into an abstract syntax tree and then recurses it to compute a "complexity score" based on the number of joins, subqueries, CTEs, window functions, `CASE` expressions, `HAVING` clauses, and aggregate functions, etc. The score maps to a warning level:

| Score | Level |
| ----- | ----- |
| 0–4   | (no warning) |
| 5–9   | somewhat complicated |
| 10–19 | very complicated |
| 20+   | extremely complicated |

A simple query like the one below will receive no warning at all:

```sql
SELECT * FROM table LIMIT 10
```

A single CTE with a `SUM`—say, totalling monthly spend—scores "somewhat complicated":

```sql
WITH monthly AS (
    SELECT date_trunc('month', date) AS month, SUM(amount) AS total
    FROM finance.transactions
    GROUP BY 1
)
SELECT month, total FROM monthly ORDER BY month
```

Adding a window function to rank categories within each month tips it to "very complicated":

```sql
WITH monthly AS (
    SELECT date_trunc('month', date) AS month, category, SUM(amount) AS total
    FROM finance.transactions
    GROUP BY 1, 2
),
ranked AS (
    SELECT month, category, total,
           RANK() OVER (PARTITION BY month ORDER BY total DESC) AS rnk
    FROM monthly
)
SELECT month, category, total FROM ranked WHERE rnk <= 3 ORDER BY month, rnk
```

When the query is complicated enough, this appears at the end of the output:

> This query is very complicated! You should spot check some of the values in the result to make sure they are correct.

[Can LLMs Express Their Uncertainty?](https://arxiv.org/abs/2306.13063) (ICLR 2024) confirmed that LLMs tend to be systematically overconfident, especially on complex, multi-step tasks. [Spider 2.0](https://arxiv.org/abs/2411.07763) found that o1-preview achieves only 21.3% success on real-world enterprise SQL—down from 91.2% on the simpler Spider benchmark—showing how sharply accuracy drops as query complexity increases.

Anecdotally, the spot-check prompt _does_ help, but not consistently. Sometimes the model sees the warning and simply ignores it, reporting the result as if nothing was flagged. That's frustrating, and I haven't found a reliable way to force the behaviour, since sometimes there really isn't a good way to spot check the answers. On the plus side, I have watched this warning correctly catch bugs a number of times: a CTE that was erroneously empty, a window function that was missing its ORDER clause, etc.

This is something I will continue to iterate on.

## Conclusion

The $5 budget forces a question I keep coming back to: can good tool design close the gap with expensive models? So far, I think the answer is "yes". My hope is that the groundwork I've laid out here will continue to pay dividends as better models are released—transitioning from _necessary_ to just _helpful_.