# MySQL

**1.Start the MySQL Container:**

Run the official MySQL container with your specific credentials (`root` password: `manav`, database: `manav`) bound to port `3306`:

Bash

```
docker run -d \
  --name mysql-books \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=manav \
  -e MYSQL_DATABASE=manav \
  mysql:8.0
```

**2.Create the Table and Seed Sample Data:**

{% code title="" overflow="wrap" lineNumbers="true" expandable="true" %}
```
docker exec -i mysql-books mysql -uroot -pmanav manav <<EOF
DROP TABLE IF EXISTS books;
DROP TABLE IF EXISTS authors;
DROP TABLE IF EXISTS categories;

CREATE TABLE authors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    country VARCHAR(50) NOT NULL
);

CREATE TABLE categories (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL
);

CREATE TABLE books (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    price DECIMAL(6,2) NOT NULL,
    author_id INT,
    category_id INT,
    published_year INT,
    FOREIGN KEY (author_id) REFERENCES authors(id) ON DELETE SET NULL,
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE SET NULL
);
EOF
```
{% endcode %}

**3.Seed 10,000 Records via Stored Procedure:**<br>

{% code title="" overflow="wrap" lineNumbers="true" expandable="true" %}
```
docker exec -i mysql-books mysql -uroot -pmanav manav <<EOF
-- Seed Base Categories
INSERT INTO categories (name) VALUES 
('Programming'), ('Science Fiction'), ('Databases'), ('DevOps'), 
('Design'), ('History'), ('Philosophy'), ('Business'), ('Networking'), ('Security');

-- Seed 100 Authors
DELIMITER $$
CREATE PROCEDURE SeedAuthors()
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i <= 100 DO
        INSERT INTO authors (name, country) 
        VALUES (CONCAT('Author_', i), ELT(1 + FLOOR(RAND()*4), 'USA', 'India', 'UK', 'Germany'));
        SET i = i + 1;
    END WHILE;
END$$
DELIMITER ;

CALL SeedAuthors();
DROP PROCEDURE SeedAuthors;

-- Seed 10,000 Books
DELIMITER $$
CREATE PROCEDURE SeedBooks()
BEGIN
    DECLARE i INT DEFAULT 1;
    START TRANSACTION;
    WHILE i <= 10000 DO
        INSERT INTO books (title, price, author_id, category_id, published_year) 
        VALUES (
            CONCAT('Book Title #', i),
            ROUND(10 + (RAND() * 90), 2),
            FLOOR(1 + (RAND() * 100)),
            FLOOR(1 + (RAND() * 10)),
            FLOOR(2000 + (RAND() * 26))
        );
        SET i = i + 1;
    END WHILE;
    COMMIT;
END$$
DELIMITER ;

CALL SeedBooks();
DROP PROCEDURE SeedBooks;
EOF
```
{% endcode %}

**4.Test Your SQL Joins:**<br>

{% code title="" overflow="wrap" lineNumbers="true" expandable="true" %}
```
# Check total count
docker exec -i mysql-books mysql -uroot -pmanav manav -e "SELECT COUNT(*) AS total_books FROM books;"

# Run an INNER JOIN across all 3 tables
docker exec -i mysql-books mysql -uroot -pmanav manav -e "
SELECT 
    b.id, 
    b.title, 
    b.price, 
    a.name AS author, 
    a.country, 
    c.name AS category 
FROM books b
JOIN authors a ON b.author_id = a.id
JOIN categories c ON b.category_id = c.id
LIMIT 5;
"
```
{% endcode %}
