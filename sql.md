SELECT * FROM users;

SELECT name FROM users;

SELECT name, age FROM users;

SELECT * FROM users WHERE age > 30;

SELECT * FROM users WHERE age >= 20 AND age <= 30;

SELECT * FROM users WHERE city = 'Tokyo';

SELECT * FROM users WHERE city != 'Tokyo';

SELECT * FROM users WHERE city IN ('Tokyo', 'Osaka');

SELECT * FROM users WHERE city NOT IN ('Tokyo', 'Osaka');

SELECT * FROM users WHERE age BETWEEN 20 AND 30;

SELECT * FROM users WHERE name LIKE 'A%';

SELECT * FROM users WHERE name LIKE '%a%';

SELECT * FROM users WHERE city IS NULL;

SELECT * FROM users WHERE city IS NOT NULL;

SELECT * FROM users ORDER BY age;

SELECT * FROM users ORDER BY age DESC;

SELECT * FROM users ORDER BY city, age DESC;

SELECT * FROM users LIMIT 5;

SELECT * FROM users ORDER BY age DESC LIMIT 3;

SELECT DISTINCT city FROM users;

SELECT COUNT(*) FROM users;

SELECT COUNT(*) FROM users WHERE age >= 30;

SELECT AVG(age) FROM users;

SELECT MAX(age) FROM users;

SELECT MIN(age) FROM users;

SELECT SUM(price) FROM products;

SELECT city, COUNT(*) FROM users GROUP BY city;

SELECT city, AVG(age) FROM users GROUP BY city;

SELECT city, COUNT(*) FROM users GROUP BY city HAVING COUNT(*) >= 2;

SELECT category, AVG(price) FROM products GROUP BY category;

SELECT category, MAX(price) FROM products GROUP BY category;

SELECT * FROM products WHERE price > (SELECT AVG(price) FROM products);

SELECT * FROM users WHERE age = (SELECT MAX(age) FROM users);

SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE city = 'Tokyo');

SELECT users.name, orders.id
FROM users
JOIN orders ON users.id = orders.user_id;

SELECT users.name, orders.total
FROM users
JOIN orders ON users.id = orders.user_id
WHERE orders.total >= 10000;

SELECT users.name, COUNT(orders.id)
FROM users
LEFT JOIN orders ON users.id = orders.user_id
GROUP BY users.id, users.name;

SELECT products.name, categories.name
FROM products
JOIN categories ON products.category_id = categories.id;

SELECT users.name, SUM(orders.total)
FROM users
JOIN orders ON users.id = orders.user_id
GROUP BY users.id, users.name;

SELECT users.name, SUM(orders.total) AS spending
FROM users
JOIN orders ON users.id = orders.user_id
GROUP BY users.id, users.name
ORDER BY spending DESC;

SELECT
    name,
    age,
    CASE
        WHEN age < 20 THEN 'young'
        WHEN age < 40 THEN 'adult'
        ELSE 'senior'
    END AS group_name
FROM users;

SELECT name, COALESCE(city, 'Unknown') FROM users;

SELECT name, ROUND(price, 0) FROM products;

SELECT name, LENGTH(name) FROM users;

SELECT UPPER(name) FROM users;

SELECT LOWER(name) FROM users;

SELECT * FROM orders
WHERE date(created_at) >= date('now', '-30 days');

SELECT strftime('%Y-%m', created_at) AS month, COUNT(*)
FROM orders
GROUP BY month
ORDER BY month;

WITH expensive AS (
    SELECT *
    FROM products
    WHERE price >= 10000
)
SELECT * FROM expensive;

SELECT
    name,
    price,
    ROW_NUMBER() OVER (ORDER BY price DESC) AS rank
FROM products;
