# Library_Checkout_Trends_Analysis

This was an independent project I developed using the Seattle Public Library's open dataset [Checkouts by Title](https://data.seattle.gov/Community/Checkouts-by-Title/tmmm-ytt6), a monthly count by title of every item borrowed from the library since 2005. The dataset currently has 42.5 million rows.

I used SQL in the BigQuery workspace to explore and analyse the data. My primary aim was to compare physical and digital book borrowing trends to help library decision-makers forecast user demand for different formats. I compared the checkout numbers over time among four book formats – physical books, ebooks, physical audiobooks, and digital audiobooks – and also identified the top 5 most borrowed titles per year for each format. I found that ebooks and digital audiobooks have both had a high rate of growth since 2011 and now make up 61% of all book loans. Unlike physical books, digital book checkout trends were not strongly affected by the COVID-19 pandemic and demand has continued on an upward trend.

Challenges included finding the top 5 performing items in a category over multiple years, and returning multiple sums based on different criteria. Some of the more advanced techniques I used were nested subqueries, partitioning, row numbers, and complex aggregating and grouping.
 
Once my data was extracted, I used Tableau to create an interactive visualization, and published an article on LinkedIn. The dashboard and article were both aimed at a general audience and intended to share the story I found in the data more widely.

[Dashboard on Tableau Public](https://public.tableau.com/app/profile/jessica.lamothe/viz/SPLCheckoutsViz/Dashboard1)

[Article on LinkedIn](https://www.linkedin.com/pulse/rise-digital-library-books-17-years-data-from-seattle-jessica-lamothe)

## Table of Contents

[Business Questions](https://github.com/jessicalamothe/Library_Checkout_Trends_Analysis#business-questions)

[Data Exploration and Cleaning](https://github.com/jessicalamothe/Library_Checkout_Trends_Analysis#data-exploration-and-cleaning)

[Data Analysis](https://github.com/jessicalamothe/Library_Checkout_Trends_Analysis#data-analysis)

[Tables Created](https://github.com/jessicalamothe/Library_Checkout_Trends_Analysis#tables-created)

[Dashboard](https://github.com/jessicalamothe/Library_Checkout_Trends_Analysis#dashboard)

[Conclusions and Recommendations](https://github.com/jessicalamothe/Library_Checkout_Trends_Analysis#conclusions-and-recommendations)

## Business Questions

The shift to digital book formats, including both ebooks and digital audiobooks, is a major ongoing change in the library and publishing sectors. Digital readership is growing: In 2021, 39% of adult American readers used ebooks and 31% used audiobooks, up from 22% and 14% in 2011, while printed book use dropped from 93% to 85% in the same period.[^1]

[^1]: Pew Research Centre survey, Jan. 25-Feb. 8, 2021

With this in mind, I chose to focus my analysis on the following questions:

**1. How many physical books and audiobooks were borrowed per year, and how many digital ebooks and audiobooks?**

**2. How were checkout numbers for each of the four book formats affected by the COVID-19 pandemic?**

**3. What were the top 5 books borrowed per year for each format?**

The primary audience envisaged was the Seattle Public Library’s leadership team. By identifying trends in user demand for different book formats, this analysis aimed to inform strategic planning, allowing leaders to efficiently invest in collections to meet the current and forecasted demand.

In addition, I wanted to create a dashboard that was of general public interest, to share the story this data tells us about changing book readership.

## Data Exploration and Cleaning

The data consisted of:

- 12 columns, including: Title, MaterialType, CheckoutYear, CheckoutMonth, Checkouts (a count of the number of times an item was checked out within the checkout month).
- 42.5 million rows.

I used SQL in the Google BigQuery workspace to review, clean, and analyse this large dataset. I downloaded the dataset then uploaded it into Google Cloud Storage to use with BigQuery.

Data cleaning process:

- Use SELECT DISTINCT on the MaterialType field to show the unique values. This would reveal any null values or formatting inconsistencies.  It was important to establish that this field was used consistently, because I would be using it to identify the different book formats. 

```
SELECT DISTINCT MaterialType, Count(*) AS number_of_uses
FROM `spl-checkouts`
GROUP BY MaterialType
ORDER BY number_of_uses DESC;
```

Result: field appears to be used consistently, with no null values. There are 71 values describing various items loaned by the library, from BOOK (22378286) to ER, VIDEOCASS (1). I am interested in the values BOOK, REGPRINT, and LARGEPRINT for physical books (these will need to be combined into one category), EBOOK for ebooks, AUDIOBOOK for digital audiobooks, and SOUNDDISC, SOUNDCASS, and SOUNDREC for physical audiobooks (on CD or cassette). This last category also includes music recordings, which I will need to separate from the audiobooks.

- Use SELECT DISTINCT on CheckoutYear and CheckoutMonth. Result: CheckoutYear includes only integers from 2005-2023 and CheckoutMonth includes only integers from 1-12. The data ranges from April 2005-March 2023. My analysis will exclude the years 2005 and 2023 because they have incomplete data.

- Use SELECT DISTINCT on UsageClass and CheckoutType to check for distinct values: no nulls.

- Use COUNT DISTINCT to count distinct values in Title: 1846835. Use IS NULL to check for null values in Title: none.

- Examine the numerical values in Checkouts (the number of times an item was checked out per month). Min: 1, Max: 4903, second highest: 2950, average: 3.48. I examined the top 20 and found that the highest few checkout numbers were outliers. Most were headphones and laptops, but interestingly, nine were ebooks and audiobooks. The top borrowed item in a single month was So You Want to Talk about Race, with 4903 	checkouts in June 2020. Research confirmed that these outlying ebooks and audiobooks were in a special ‘always available’ format, allowing multiple users to borrow them simultaneously.

## Data Analysis

### Part 1: Number of Loans Per Year by Material Type

I wanted to find the total number of checkouts per year for four material types: physical books, ebooks, physical audiobooks, and digital audiobooks.

First, I wrote a simple query to find the sum of monthly checkouts grouped by checkout year and material type, while filtering to return only the material types I am interested in (excluding physical audiobooks, which I will return to) and to exclude 2005 and 2023.

```
SELECT SUM(Checkouts) AS total_no_checkouts, CheckoutYear, MaterialType
FROM `spl-checkouts`
WHERE CheckoutYear NOT IN (2005, 2023) AND (MaterialType = 'BOOK' OR MaterialType = 'EBOOK' OR MaterialType = 'AUDIOBOOK' OR MaterialType = 'REGPRINT' OR MaterialType = 'LARGEPRINT')
GROUP BY CheckoutYear, MaterialType
ORDER BY CheckoutYear, MaterialType;
```

This resulted in a table of sums by year and book format. However, I still needed to combine the sums for BOOK,  REGPRINT and LARGEPRINT into one sum for physical books.

I used my first table as a subquery and filtered it to show only the physical book material types:

```
SELECT SUM(total_no_checkouts) AS physical_book_checkouts, CheckoutYear
FROM
(
SELECT SUM(Checkouts) AS total_no_checkouts, CheckoutYear, MaterialType
FROM `spl-checkouts`
WHERE CheckoutYear NOT IN (2005, 2023) AND (MaterialType = 'BOOK' OR MaterialType = 'REGPRINT' OR MaterialType = 'LARGEPRINT' OR MaterialType = 'EBOOK' OR MaterialType = 'AUDIOBOOK')
GROUP BY CheckoutYear, MaterialType
ORDER BY CheckoutYear, MaterialType
)
WHERE MaterialType = 'BOOK' OR MaterialType = 'REGPRINT' OR MaterialType = 'LARGEPRINT'
GROUP BY CheckoutYear
ORDER BY CheckoutYear;
```

This gave me the sums I needed. However, I wanted to find out if I could write a single query to return the sums for physical books in the same table as audiobooks and ebooks.

The problem was that I needed to select multiple sums from my first table based on different criteria within the MaterialType column. I Googled how to select multiple sums in a single SQL statement, and found a StackOverflow answer which reminded me of the CASE WHEN command. This enabled me to write the following query to return the sums for all three book formats:

```
SELECT
SUM(CASE WHEN (MaterialType = 'BOOK' OR MaterialType = 'REGPRINT' OR MaterialType = 'LARGEPRINT') THEN total_no_checkouts END) AS physical_book_checkouts,
SUM(CASE WHEN MaterialType = 'EBOOK' THEN total_no_checkouts END) AS ebook_checkouts,
SUM(CASE WHEN MaterialType = 'AUDIOBOOK' THEN total_no_checkouts END) AS audiobook_checkouts,
CheckoutYear
FROM
(
SELECT SUM(Checkouts) AS total_no_checkouts, CheckoutYear, MaterialType
FROM `spl-checkouts`
WHERE CheckoutYear NOT IN (2005, 2023) AND (MaterialType = 'BOOK' OR MaterialType = 'REGPRINT' OR MaterialType = 'LARGEPRINT' OR MaterialType = 'EBOOK' OR MaterialType = 	'AUDIOBOOK')
GROUP BY CheckoutYear, MaterialType
ORDER BY CheckoutYear, MaterialType
)
GROUP BY CheckoutYear
ORDER BY CheckoutYear;
```

For the final step, I needed to extract the checkout data on physical audiobooks. The MaterialType value for these was either SOUNDDISC, SOUNDCASS, and SOUNDREC, but these categories also included recordings of music, which I wanted to filter out for the purposes of my analysis.

By studying the data I found I could filter out the music using the Subjects field, which displays the library’s classification tags. For music recordings, the subjects included recurring words such as ‘music’ and ‘piano’, so I filtered out a number of these in the WHERE clause, eg: `Subjects NOT LIKE '%music%'`. This did not remove every music recording, but by randomly sampling the data after applying this filter I estimated music to make up less than 1% of the results.

I inserted this filter into my first query to find the number of physical audiobook checkouts, and used Excel to add the results to my table of checkouts per year.

### Part 2: Top Titles Per Year

For the next part of my analysis, I wanted to find the top 5 most checked out titles per year for each of the four material types.

I started with a query to find the sum of the monthly checkouts by title for a single material type and year (also filtering out some records where the book title was missing), limited to the top 5 results:

```
SELECT Sum(Checkouts) AS total_checkouts_that_year, Title
FROM `spl-checkouts`
WHERE (MaterialType = 'BOOK' OR MaterialType = 'REGPRINT' OR MaterialType = 'LARGEPRINT') AND CheckoutYear = 2007 AND TITLE NOT LIKE '%Unknown%' AND TITLE NOT LIKE '%Uncataloged%'
GROUP BY Title
ORDER BY total_checkouts_that_year DESC
LIMIT 5;
```

The WHERE clause could be altered to find results for different years and material types. However, this would require manually running the query many times, so I wanted to see if I could write a query to return multiple years at once.

After experimenting and searching online, I figured out that I would need to:

1. Create a table to specify a single material type (such as physical book) and aggregate the total checkouts by title and year. This creates a long, unsorted list of titles, years, and the number of times the title was checked out that year.

2. Using the first table as a subquery, partition it by year, order each partition by total checkouts, and apply row numbers. This orders the table into descending lists of the most checked out titles per year, with each year having separate row numbers.

3. Filter the second table to return only row numbers 1-5. This returns the top 5 checked out titles for each year.

```
SELECT
total_checkouts_that_year,
Title,
CheckoutYear
FROM
(
SELECT
total_checkouts_that_year,
Title,
CheckoutYear,
ROW_NUMBER() OVER (PARTITION BY CheckoutYear ORDER BY total_checkouts_that_year DESC) RN
FROM
(
SELECT
Sum(Checkouts) AS total_checkouts_that_year,
Title,
CheckoutYear
FROM `spl-checkouts`
WHERE (MaterialType = 'BOOK' OR MaterialType = 'REGPRINT' OR MaterialType = 'LARGEPRINT') AND TITLE NOT LIKE '%Unknown%' AND TITLE NOT LIKE '%Uncataloged%' AND CheckoutYear NOT IN (2005, 2023)
GROUP BY CheckoutYear, Title
ORDER BY total_checkouts_that_year DESC
)
)
WHERE
RN <6
ORDER BY CheckoutYear DESC, RN ASC;
```

## Tables Created

(1) Total checkouts per year of physical books, ebooks, digital audiobooks, and physical audiobooks from 2006 to 2022

(2) Top 5 most checked out books across the four formats from 2006 to 2022

[See tables here](https://github.com/jessicalamothe/Library_Checkout_Trends_Analysis/blob/main/library_checkout_analysis_tables.xlsx)

## Dashboard

![The Rise of Digital Library Books: 17 Years of Data from the Seattle Public Library](https://github.com/jessicalamothe/Library_Checkout_Trends_Analysis/assets/144539075/90b8890c-5e88-4067-8053-b2dcd4cef865)

[Dashboard on Tableau Public](https://public.tableau.com/app/profile/jessica.lamothe/viz/SPLCheckoutsViz/Dashboard1)

Aim: to create an interactive visualization aimed at a general audience to share the story I found in the data.

I uploaded the tables created through my analysis into Tableau Desktop and created the dashboard linked above.

The visual focal point is a set of four icons I created to represent the four book formats. Clicking on these highlights the corresponding line charting checkout numbers over time.

Below this, there is a text table showing the top 5 borrowed titles per year for each book format. Viewers can filter by year using the drop-down.

## Conclusions and Recommendations

Conclusions:

- Digital ebooks and audiobooks have both had a high rate of growth at the Seattle Public Library since 2011. In 2022, digital books made up 61% of all book loans.

- Total numbers of book checkouts have increased 110% since 2006, mainly due to the influx of digital books.

- In 2006, 93% of all books borrowed were physical books, 7% were ebooks, and less than 1% were digital ebooks or audiobooks. In 2022, only 38% of all books borrowed were physical books and 1% were physical audiobooks, while 36% were ebooks and 25% were digital audiobooks.

- COVID-19 library visiting restrictions in 2020 and 2021 resulted in steep declines of physical book and audiobook checkouts. This effect is still being felt: in 2022 checkout numbers for both were still lower than in any previous year in the data.

- Conversely, the pandemic did not have a strong positive effect on digital book loans, as might have been expected. Ebook checkouts showed a slight increase in growth rate in 2020, but both were already growing at a high rate even before COVID-19.

- Physical book checkouts increased from 2006 to 2009, then declined slightly through the 2010s. The demand for physical books is currently lower than before COVID-19, but 2022 rates increased 27% from 2021.

- Similar titles appear across the lists of most borrowed books compared by format. The top 5 ebooks and audiobooks are now typically borrowed twice as often as the top physical books.

The limitations of this data should also be noted:

- This data represents only one library in the USA, and its borrowers have specific demographics and are not representative of all library users or book consumers. Specific book popularity is limited to which items are in the library’s collection and affected by local factors such as book clubs.

- Further data from the SPL would be necessary for a holistic understanding of borrowing trends. For example, was the rise in digital book use driven only by user demand, or was it affected by deliberate investments or marketing efforts?

Recommendations:

Based on this data alone, I would recommend that library decision-makers increase investment in digital ebooks and audiobooks in upcoming years to meet the demand that the checkout trends predict.

The demand for physical books is currently lower than before COVID-19, but 2022 rates increased 27% from 2021. Close monitoring of the next few years is necessary to discover whether, once physical book demand recovers from the pandemic, it resumes its previous gradual decline.

There has been a significant decline in demand for physical audiobooks, accelerate by COVID-19, so investment in this format may be lowered.
