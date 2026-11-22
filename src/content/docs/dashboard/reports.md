---
title: Reports
description: The Reports screen compares chosen links side by side over a period, with filters such as country and platform, a chart with a line per link, a table, a split by any dimension, CSV, and saved reports the team can open again.
sidebar:
  order: 2
---

The Reports screen, at https://dashboard.adto.app/dashboard/reports, compares the links you choose. Its header reads "Compare links side by side". A report is saved under a name, and opening it again runs it on fresh numbers. The links list's **Compare** opens a report too (see [Links](/dashboard/links/#compare)).

## The list

The list shows each saved report you may see:

| Column | What it shows |
|---|---|
| **Report** | Its name, which opens it, and **Private** when only its creator sees it |
| **Links** | How many links it compares |
| **Period** | Its period, such as **Last 30 days** |
| **Made by** | **You**, or the teammate's email |
| **Changed** | When it was last saved |

**New report** opens an empty one. On Free and Starter, a count such as `2 of 3 saved` shows how many reports the plan keeps (Free 3, Starter 10; Growth and Scale any number). With the plan's reports all saved, a notice says so. Comparing links without saving still works.

## A report

### What it compares

**Links (n of 100).** **Choose links** opens a list of the account's links with a search by slug, label or campaign, and three shortcuts:
- **Top 10 by installs** and **Top 10 by clicks**, over the last year, temporary links included;
- **Every link of a campaign…**.

Temporary links are listed too. **Show deleted links** adds deleted ones, which keep their numbers. A chosen link shows as a chip with its colour on the chart; × removes it.

**Period:**
- **Today** or **Yesterday**, charted by the hour;
- **Last 7**, **30**, **90** or **365 days**, **This month** or **Last month**;
- **Custom dates…**, with a start and an end day (at most 366 days).

A preset is counted from the day the report runs, so a saved **Last 30 days** always shows the latest 30 days. The days follow the report's time zone, which is the zone of the person who made it (shown under its name), so everyone on the team sees the same numbers.

**Metrics:** **Clicks**, **Installs** (first installs), **Reinstalls**, **App opens**, **Leads**, **Purchases** (a refund comes off the revenue, not the count), **Revenue** and **LTV** (net revenue to date per first install of the period). Click one to show or hide it; at least one stays on.

**Filters:** **Add filter** opens two pickers.
- **The first picks what to filter:** platform, country, campaign, a UTM, city, OS version, channel, install store, or any other of [the analytics dimensions](/api/admin-api/#the-analytics-query). Type to search.
- **The second picks its values,** several at once. It lists the values seen in the last year, most taps first:
  - countries by name with their codes;
  - platforms as iOS, Android, Desktop and Other;
  - **Not set** for links that carried no value.

  For a value not seen yet, such as a campaign about to start, type it in. **Done** closes the list; **Add** applies the filter.

A filter keeps only the values chosen. Click its chip to change it; × removes it. A filter made through the API can leave values out instead; its chip then reads "not".

**Split the links by**, a picker of the same dimensions, adds a table of each link split by one dimension, such as platform or country.

### What it shows

- **Totals:** each metric over all the report's links, and **Conversion** (first installs ÷ clicks) when both are shown.
- **The chart:** one line per link for one metric; the tabs above it switch the metric. It draws the 10 links with the most of that metric; the others are grey on the chips and the table, and a note gives their sum.
- **The table:** one row per link with each metric, **Conv.** and **Share** (the link's part of the first metric that adds up across links; LTV, an average per install, has none, so a report of LTV alone has no Share).
  - A column header sorts by it.
  - Shading marks the larger numbers in each column.
  - The last row totals every link.
  - A link with nothing in the period shows zeros.
- **By {dimension}:** the split, each link's rows together.

**CSV** downloads the table.

### Saving and sharing

- **Save** stores the report. A new one needs a name, typed in its title.
- **Team** or **Only me** sets who sees it. A team report is everyone's in the account. Only its creator sees a private one, and the owner sees it listed and may delete it.
- **Save as copy** saves the settings as a new report of your own.
- **Delete** is there for the report's creator and for the owner.

**Who can do what:**
- **Viewers** and **support sessions** open and change a report's settings to look, but do not save.
- **Members** save, and edit their own and team reports.
- **Only the creator** changes who sees one.

The numbers come from the same queries as the Overview's [Breakdown](/dashboard/overview/), so they agree with it for the same period and filters. A period that starts before the data your account keeps says so above the totals.
