# Netflix Advanced SQL Project

![Netflix Logo]([https://img.shields.io/badge/NETFLIX-E50914?style=for-the-badge&logo=netflix&logoColor=white](https://github.com/ImTaRunGupTa/Netlfix_Data_Analysis_SQL/blob/main/netflix-logo.avif))


[Click Here to get Dataset](https://www.kaggle.com/datasets/shivamb/netflix-shows)

## Overview
This project analyses the Netflix catalogue (movies and TV shows) using **SQL**. It covers an end-to-end process of creating the schema, cleaning multi-value columns (genres, cast, country), handling text dates and durations, and answering business questions of varying complexity (easy, medium and advanced). The primary goals are to practise advanced SQL skills and generate insights about what content Netflix adds, where it comes from and how its catalogue grows.

## Project Steps
1. **Data Exploration**: understand the columns and spot the messy ones (multi-value text, dates stored as text, missing values).
2. **Schema and Load**: create the table(s) and import the CSV.
3. **Querying the Data**: solve the 14 questions below in order (**easy**, **medium**, **advanced**).
4. **Insights and Visualization**: summarise findings and optionally build a dashboard.

---

## Schema
```sql
DROP TABLE IF EXISTS netflix;
CREATE TABLE netflix (
    show_id      VARCHAR(10),
    type         VARCHAR(10),
    title        VARCHAR(250),
    director     VARCHAR(550),
    casts        VARCHAR(1050),
    country      VARCHAR(550),
    date_added   VARCHAR(55),   -- stored as text, e.g. 'September 25, 2021'
    release_year INT,
    rating       VARCHAR(15),
    duration     VARCHAR(15),   -- '90 min' or '2 Seasons'
    listed_in    VARCHAR(250),  -- comma separated genres
    description  VARCHAR(550)
);
```

## Data Exploration
- `type`: Movie or TV Show.
- `country`, `casts`, `director`, `listed_in`: multi-value columns separated by commas (use `STRING_TO_ARRAY` + `UNNEST`).
- `date_added`: text, convert with `TO_DATE(TRIM(date_added), 'Month DD, YYYY')`.
- `duration`: minutes for movies, seasons for TV shows (use `SPLIT_PART(duration, ' ', 1)::INT`).

## 14 Practice Questions

### Easy Level
1. Count the number of Movies vs TV Shows.
```sql
SELECT type, COUNT(*) AS total_content
FROM netflix
GROUP BY type;
```
2. List all movies released in 2020.
```sql
SELECT title, release_year
FROM netflix
WHERE type = 'Movie' AND release_year = 2020;
```
3. Count the number of titles for each rating.
```sql
SELECT rating, COUNT(*) AS total_titles
FROM netflix
GROUP BY rating
ORDER BY total_titles DESC;
```
4. Find all movies longer than 150 minutes.
```sql
SELECT title, duration
FROM netflix
WHERE type = 'Movie'
  AND SPLIT_PART(duration, ' ', 1)::INT > 150
ORDER BY SPLIT_PART(duration, ' ', 1)::INT DESC;
```
5. Find all titles directed by 'Rajiv Chilaka'.
```sql
SELECT title, type, release_year
FROM netflix
WHERE director ILIKE '%Rajiv Chilaka%';
```

### Medium Level
6. Find the top 5 countries with the most content on Netflix.
```sql
SELECT TRIM(UNNEST(STRING_TO_ARRAY(country, ','))) AS country,
       COUNT(*) AS total_content
FROM netflix
WHERE country IS NOT NULL
GROUP BY 1
ORDER BY total_content DESC
LIMIT 5;
```
7. Find the most common rating for Movies and for TV Shows.
```sql
SELECT type, rating
FROM (
    SELECT type, rating, COUNT(*) AS cnt,
           RANK() OVER (PARTITION BY type ORDER BY COUNT(*) DESC) AS rnk
    FROM netflix
    GROUP BY type, rating
) t
WHERE rnk = 1;
```
8. List all content added to Netflix in the last 5 years.
```sql
SELECT title, type, TO_DATE(TRIM(date_added), 'Month DD, YYYY') AS added_on
FROM netflix
WHERE date_added IS NOT NULL
  AND TO_DATE(TRIM(date_added), 'Month DD, YYYY') >= CURRENT_DATE - INTERVAL '5 years'
ORDER BY added_on DESC;
```
9. Find the top 10 actors who appeared in the highest number of Indian movies.
```sql
SELECT TRIM(UNNEST(STRING_TO_ARRAY(casts, ','))) AS actor,
       COUNT(*) AS total_movies
FROM netflix
WHERE country ILIKE '%India%' AND type = 'Movie'
GROUP BY 1
ORDER BY total_movies DESC
LIMIT 10;
```
10. Categorise content as 'Bad' if the description contains the words 'kill' or 'violence', otherwise 'Good', and count each category.
```sql
SELECT category, COUNT(*) AS total_content
FROM (
    SELECT CASE
             WHEN description ILIKE '%kill%' OR description ILIKE '%violence%' THEN 'Bad'
             ELSE 'Good'
           END AS category
    FROM netflix
) t
GROUP BY category;
```

### Advanced Level
11. For each year content was added, calculate the share of that year in all Indian content on Netflix.
```sql
SELECT EXTRACT(YEAR FROM TO_DATE(TRIM(date_added), 'Month DD, YYYY'))::INT AS year_added,
       COUNT(*) AS india_content,
       ROUND(COUNT(*)::NUMERIC /
            (SELECT COUNT(*) FROM netflix WHERE country ILIKE '%India%') * 100, 2) AS pct_of_india_content
FROM netflix
WHERE country ILIKE '%India%' AND date_added IS NOT NULL
GROUP BY 1
ORDER BY 1;
```
12. Find the longest movie in each genre using a CTE and window function.
```sql
WITH genre_movies AS (
    SELECT TRIM(UNNEST(STRING_TO_ARRAY(listed_in, ','))) AS genre,
           title,
           SPLIT_PART(duration, ' ', 1)::INT AS minutes
    FROM netflix
    WHERE type = 'Movie' AND duration IS NOT NULL
),
ranked AS (
    SELECT genre, title, minutes,
           RANK() OVER (PARTITION BY genre ORDER BY minutes DESC) AS rnk
    FROM genre_movies
)
SELECT genre, title, minutes
FROM ranked
WHERE rnk = 1
ORDER BY minutes DESC;
```
13. Find directors who have made both Movies and TV Shows.
```sql
WITH directors AS (
    SELECT TRIM(UNNEST(STRING_TO_ARRAY(director, ','))) AS director, type
    FROM netflix
    WHERE director IS NOT NULL
)
SELECT director,
       COUNT(*) FILTER (WHERE type = 'Movie')   AS movies,
       COUNT(*) FILTER (WHERE type = 'TV Show') AS tv_shows
FROM directors
GROUP BY director
HAVING COUNT(DISTINCT type) = 2
ORDER BY movies + tv_shows DESC;
```
14. Calculate year-over-year growth in the number of titles added to Netflix using `LAG`.
```sql
WITH yearly AS (
    SELECT EXTRACT(YEAR FROM TO_DATE(TRIM(date_added), 'Month DD, YYYY'))::INT AS year_added,
           COUNT(*) AS total
    FROM netflix
    WHERE date_added IS NOT NULL
    GROUP BY 1
)
SELECT year_added,
       total,
       LAG(total) OVER (ORDER BY year_added) AS previous_year,
       ROUND((total - LAG(total) OVER (ORDER BY year_added))::NUMERIC
             / NULLIF(LAG(total) OVER (ORDER BY year_added), 0) * 100, 2) AS yoy_growth_pct
FROM yearly
ORDER BY year_added;
```

---

## Technology Stack
- **Database**: PostgreSQL
- **SQL Concepts**: DDL, DML, Aggregations, Subqueries, CTEs, Window Functions (`RANK`, `LAG`), `UNNEST` / `STRING_TO_ARRAY`, Date functions, `CASE`, `FILTER`
- **Tools**: pgAdmin 4 (or any SQL editor), PostgreSQL (via Docker, Homebrew or direct installation)

## How to Run the Project
1. Install PostgreSQL and pgAdmin (if not already installed).
2. Create a database, e.g. `CREATE DATABASE streaming_sql;`.
3. Download the dataset from the link at the top and run the `CREATE TABLE` script above.
4. Import the data:
   ```sql
   \copy netflix FROM 'netflix_titles.csv' WITH (FORMAT csv, HEADER true);
   ```
5. Execute the queries in order and compare the results with your own exploratory analysis.
6. Explore indexing and query optimisation techniques.

## Key Insights You Can Report
- Movies vs TV show split and the most common ratings.
- Top content-producing countries and most frequent actors in Indian movies.
- How the number of titles added per year has grown (year-over-year).
- Which genres have the longest movies.

---

## Next Steps
- **Visualize the Data**: Build a Power BI or Tableau dashboard (content by country, genre heatmap, titles added per year).
- **Normalise**: Split `listed_in`, `casts` and `country` into bridge tables instead of using `UNNEST` each time.
- **Optimize**: Add indexes on `type`, `release_year` and `country`, then compare `EXPLAIN ANALYZE` plans.
- **Expand**: Compare with the Prime Video and Disney+ datasets (see the cross-platform project).
