# Music-playlist-SQL

# SQL PROJECT- MUSIC STORE DATA ANALYSIS 

## Question Set 1 

1.	Who is the senior most employee based on job title? 
2.	Which countries have the most Invoices? 
3.	What are top 3 values of total invoice? 
4.	Which city has the best customers? We would like to throw a promotional Music Festival in the city we made the most money. Write a query that returns one city that has the highest sum of invoice totals. Return both the city name & sum of all invoice totals 
5.	Who is the best customer? The customer who has spent the most money will be declared the best customer. Write a query that returns the person who has spent the most money 
   
## Question Set 2 

1.	Write query to return the email, first name, last name, & Genre of all Rock Music listeners. Return your list ordered alphabetically by email starting with A 
2.	Let's invite the artists who have written the most rock music in our dataset. Write a query that returns the Artist name and total track count of the top 10 rock bands 
3.	Return all the track names that have a song length longer than the average song length. Return the Name and Milliseconds for each track. Order by the song length with the longest songs listed first 
 
## Question Set 3
1.	Find how much amount spent by each customer on artists? Write a query to return customer name, artist name and total spent 
2.	We want to find out the most popular music Genre for each country. We determine the most popular genre as the genre with the highest amount of purchases. Write a query that returns each country along with the top Genre. For countries where the maximum number of purchases is shared return all Genres 
3.	Write a query that determines the customer that has spent the most on music for each country. Write a query that returns the country along with the top customer and how much they spent. For countries where the top amount spent is shared, provide all customers who spent this amount 
 
### Question Set 1

**Q1: Who is the senior most employee based on job title?**

```sql
SELECT title, last_name, first_name
FROM employee
ORDER BY levels DESC
LIMIT 1;
```

**Q2: Which countries have the most invoices?**

```sql
SELECT billing_country, COUNT(*) AS invoice_count
FROM invoice
GROUP BY billing_country
ORDER BY invoice_count DESC;
```

**Q3: What are the top 3 values of total invoice amounts?**

```sql
SELECT total
FROM invoice
ORDER BY total DESC
LIMIT 3;
```

**Q4: Which city has the highest total invoice amount?**

```sql
SELECT billing_city, SUM(total) AS total_invoice_amount
FROM invoice
GROUP BY billing_city
ORDER BY total_invoice_amount DESC
LIMIT 1;
```

**Q5: Who is the best customer based on total spending?**

```sql
SELECT customer.customer_id, first_name, last_name, SUM(total) AS total_spending
FROM customer
JOIN invoice ON customer.customer_id = invoice.customer_id
GROUP BY customer.customer_id, first_name, last_name
ORDER BY total_spending DESC
LIMIT 1;
```

### Question Set 2 - Moderate

**Q1: List Rock music listeners ordered by email.**

**Method 1:**

```sql
SELECT DISTINCT email, first_name, last_name
FROM customer
JOIN invoice ON customer.customer_id = invoice.customer_id
JOIN invoice_line ON invoice.invoice_id = invoice_line.invoice_id
WHERE track_id IN (
    SELECT track_id
    FROM track
    JOIN genre ON track.genre_id = genre.genre_id
    WHERE genre.name = 'Rock'
)
ORDER BY email;
```

**Method 2:**

```sql
SELECT DISTINCT email AS Email, first_name AS FirstName, last_name AS LastName, genre.name AS Genre
FROM customer
JOIN invoice ON customer.customer_id = invoice.customer_id
JOIN invoice_line ON invoice.invoice_id = invoice_line.invoice_id
JOIN track ON track.track_id = invoice_line.track_id
JOIN genre ON genre.genre_id = track.genre_id
WHERE genre.name = 'Rock'
ORDER BY email;
```

**Q2: Invite the top 10 artists who have written the most Rock music.**

```sql
SELECT artist.artist_id, artist.name, COUNT(track.track_id) AS number_of_songs
FROM track
JOIN album ON track.album_id = album.album_id
JOIN artist ON album.artist_id = artist.artist_id
JOIN genre ON track.genre_id = genre.genre_id
WHERE genre.name = 'Rock'
GROUP BY artist.artist_id, artist.name
ORDER BY number_of_songs DESC
LIMIT 10;
```

**Q3: Tracks longer than the average length, ordered by length.**

```sql
SELECT name, milliseconds
FROM track
WHERE milliseconds > (
    SELECT AVG(milliseconds)
    FROM track
)
ORDER BY milliseconds DESC;
```

### Question Set 3 - Advanced

**Q1: Amount spent by each customer on the top artist.**

```sql
WITH best_selling_artist AS (
    SELECT artist.artist_id, artist.name AS artist_name, SUM(invoice_line.unit_price * invoice_line.quantity) AS total_sales
    FROM invoice_line
    JOIN track ON invoice_line.track_id = track.track_id
    JOIN album ON track.album_id = album.album_id
    JOIN artist ON album.artist_id = artist.artist_id
    GROUP BY artist.artist_id
    ORDER BY total_sales DESC
    LIMIT 1
)
SELECT c.customer_id, c.first_name, c.last_name, bsa.artist_name, SUM(il.unit_price * il.quantity) AS amount_spent
FROM invoice i
JOIN customer c ON c.customer_id = i.customer_id
JOIN invoice_line il ON il.invoice_id = i.invoice_id
JOIN track t ON t.track_id = il.track_id
JOIN album alb ON alb.album_id = t.album_id
JOIN best_selling_artist bsa ON bsa.artist_id = alb.artist_id
GROUP BY c.customer_id, c.first_name, c.last_name, bsa.artist_name
ORDER BY amount_spent DESC;
```

**Q2: Most popular music genre by country.**

**Method 1: Using CTE**

```sql
WITH popular_genre AS (
    SELECT customer.country, genre.name, COUNT(invoice_line.quantity) AS purchases,
           ROW_NUMBER() OVER (PARTITION BY customer.country ORDER BY COUNT(invoice_line.quantity) DESC) AS row_no
    FROM invoice_line
    JOIN invoice ON invoice.invoice_id = invoice_line.invoice_id
    JOIN customer ON invoice.customer_id = customer.customer_id
    JOIN track ON invoice_line.track_id = track.track_id
    JOIN genre ON track.genre_id = genre.genre_id
    GROUP BY customer.country, genre.name
)
SELECT country, name AS genre, purchases
FROM popular_genre
WHERE row_no = 1;
```

**Method 2: Using Recursive**

```sql
WITH RECURSIVE sales_per_country AS (
    SELECT customer.country, genre.name, COUNT(*) AS purchases_per_genre
    FROM invoice_line
    JOIN invoice ON invoice.invoice_id = invoice_line.invoice_id
    JOIN customer ON invoice.customer_id = customer.customer_id
    JOIN track ON invoice_line.track_id = track.track_id
    JOIN genre ON track.genre_id = genre.genre_id
    GROUP BY customer.country, genre.name
),
max_genre_per_country AS (
    SELECT country, MAX(purchases_per_genre) AS max_genre_number
    FROM sales_per_country
    GROUP BY country
)
SELECT spc.country, spc.name AS genre, spc.purchases_per_genre
FROM sales_per_country spc
JOIN max_genre_per_country mgpc ON spc.country = mgpc.country AND spc.purchases_per_genre = mgpc.max_genre_number;
```

**Q3: Top customer by spending for each country.**

**Method 1: Using CTE**

```sql
WITH Customer_with_country AS (
    SELECT customer.customer_id, first_name, last_name, billing_country, SUM(total) AS total_spending,
           ROW_NUMBER() OVER (PARTITION BY billing_country ORDER BY SUM(total) DESC) AS row_no
    FROM invoice
    JOIN customer ON invoice.customer_id = customer.customer_id
    GROUP BY customer.customer_id, first_name, last_name, billing_country
)
SELECT customer_id, first_name, last_name, billing_country, total_spending
FROM Customer_with_country
WHERE row_no = 1;
```

**Method 2: Using Recursive**

```sql
WITH RECURSIVE customer_with_country AS (
    SELECT customer.customer_id, first_name, last_name, billing_country, SUM(total) AS total_spending
    FROM invoice
    JOIN customer ON invoice.customer_id = customer.customer_id
    GROUP BY customer.customer_id, first_name, last_name, billing_country
),
country_max_spending AS (
    SELECT billing_country, MAX(total_spending) AS max_spending
    FROM customer_with_country
    GROUP BY billing_country
)
SELECT cwc.billing_country, cwc.total_spending, cwc.first_name, cwc.last_name, cwc.customer_id
FROM customer_with_country cwc
JOIN country_max_spending cms ON cwc.billing_country = cms.billing_country AND cwc.total_spending = cms.max_spending;
```
