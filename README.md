# Library-Management-System-Project

## Description

A hands-on SQL project that builds a fully functional **Library Management System** using **MySQL**. It covers everything from database design and table creation to CRUD operations, advanced queries, and stored procedures. The project is structured to progressively develop core SQL skills — making it ideal for intermediate-level learners looking to apply real-world database concepts. 

The system tracks books, members, employees, branches, and book issue/return transactions through six interrelated tables, reinforcing key concepts like foreign keys, JOINs, GROUP BY, subqueries, CTAS, and stored procedures.

---
![library](https://github.com/user-attachments/assets/fca77672-ede7-42cc-b8b3-5ac96c3e9c50)

## Project Overview

- **Title**: Library Management System
- **Level**: Intermediate
- **Database**: `library_db`
- **SQL Dialect**: MySQL

---


## Database Schema

Six core tables are created and linked via foreign keys:

| Table | Description |
|---|---|
| `branch` | Library branch locations and managers |
| `employees` | Staff details tied to branches |
| `members` | Registered library members |
| `books` | Book catalog with rental price and availability status |
| `issued_status` | Records of books issued to members by employees |
| `return_status` | Records of returned books with quality notes |
---

## Project Structure


<img width="1097" height="777" alt="Screenshot (643)" src="https://github.com/user-attachments/assets/2574202f-496b-4d40-89f2-6645f9c760bb" />

### 1. Database Setup

```sql
CREATE DATABASE library_db;
USE library_db;
```

Tables are created with appropriate data types, primary keys, and foreign key constraints to enforce referential integrity across the system.

---

### 2. CRUD Operations (Tasks 1–5)

Basic Create, Read, Update, and Delete operations on the data.

**Task 1 — Insert a New Book:**
```sql
INSERT INTO books(isbn, book_title, category, rental_price, status, author, publisher)
VALUES ('978-1-60129-456-2', 'To Kill a Mockingbird', 'Classic', 6.00, 'yes', 'Harper Lee', 'J.B. Lippincott & Co.');
```

**Task 2 — Update a Member's Address:**
```sql
UPDATE members
SET member_address = '125 Main St'
WHERE member_id = 'C101';
```

**Task 3 — Delete an Issued Record:**
```sql
DELETE FROM issued_status
WHERE issued_id = 'IS121';
```

**Task 4 — Books Issued by a Specific Employee:**
```sql
SELECT * FROM issued_status
WHERE issued_emp_id = 'E101';
```

**Task 5 — Members Who Issued More Than One Book:**
```sql
SELECT ist.issued_emp_id, e.emp_name
FROM issued_status AS ist
JOIN employees AS e ON e.emp_id = ist.issued_emp_id
GROUP BY 1, 2
HAVING COUNT(ist.issued_id) > 1;
```

---

### 3. CTAS — Create Table As Select (Task 6)

Using `CREATE TABLE AS SELECT` to generate summary tables from query results.

**Task 6 — Book Issue Count Summary:**
```sql
CREATE TABLE book_cnts AS
SELECT b.isbn, b.book_title, COUNT(ist.issued_id) AS no_issued
FROM books AS b
JOIN issued_status AS ist ON ist.issued_book_isbn = b.isbn
GROUP BY 1, 2;
```

---

### 4. Data Analysis & Findings (Tasks 7–13)

Analytical queries to extract meaningful insights from the data.

**Task 7 — Books by Category:**
```sql
SELECT * FROM books WHERE category = 'Classic';
```

**Task 8 — Total Rental Income by Category:**
```sql
SELECT b.category, SUM(b.rental_price), COUNT(*)
FROM books AS b
JOIN issued_status AS ist ON ist.issued_book_isbn = b.isbn
GROUP BY 1;
```

**Task 9 — Members Registered in the Last 180 Days:**
```sql
SELECT * FROM members
WHERE reg_date >= CURRENT_DATE - INTERVAL 180 DAY;
```

**Task 10 — Employees with Their Branch Manager:**
```sql
SELECT e1.*, b.manager_id, e2.emp_name AS manager
FROM employees AS e1
JOIN branch AS b ON b.branch_id = e1.branch_id
JOIN employees AS e2 ON b.manager_id = e2.emp_id;
```

**Task 11 — Books with Rental Price Above $7:**
```sql
CREATE TABLE books_price_greater_than_seven AS
SELECT * FROM books WHERE rental_price > 7;
```

**Task 12 — Books Not Yet Returned:**
```sql
SELECT DISTINCT ist.issued_book_name
FROM issued_status AS ist
LEFT JOIN return_status AS rs ON ist.issued_id = rs.issued_id
WHERE rs.return_id IS NULL;
```

**Task 13 — Add Book Quality Column to Return Status:**
```sql
ALTER TABLE return_status
ADD COLUMN book_quality VARCHAR(10) DEFAULT 'good';
```

---

### 5. Advanced SQL Operations (Tasks 1–6 of Advanced Section)

**Advanced Task 1 — Identify Members with Overdue Books (30+ days):**
```sql
SELECT 
    ist.issued_member_id,
    m.member_name,
    bk.book_title,
    ist.issued_date,
    DATEDIFF(CURRENT_DATE, ist.issued_date) AS over_dues_days
FROM issued_status AS ist
JOIN members AS m ON m.member_id = ist.issued_member_id
JOIN books AS bk ON bk.isbn = ist.issued_book_isbn
LEFT JOIN return_status AS rs ON rs.issued_id = ist.issued_id
WHERE rs.return_date IS NULL
  AND DATEDIFF(CURRENT_DATE, ist.issued_date) > 30
ORDER BY ist.issued_member_id;
```

**Advanced Task 2 — Update Book Status on Return (Stored Procedure):**

Automates inserting return records and flipping book availability status to `'yes'`:
```sql
DELIMITER $$
CREATE PROCEDURE add_return_records(
    IN p_return_id VARCHAR(10),
    IN p_issued_id VARCHAR(10),
    IN p_book_quality VARCHAR(10)
)
BEGIN
    DECLARE v_isbn VARCHAR(50);
    DECLARE v_book_name VARCHAR(80);

    INSERT INTO return_status(return_id, issued_id, return_date, book_quality)
    VALUES (p_return_id, p_issued_id, CURRENT_DATE, p_book_quality);

    SELECT issued_book_isbn, issued_book_name INTO v_isbn, v_book_name
    FROM issued_status WHERE issued_id = p_issued_id;

    UPDATE books SET status = 'yes' WHERE isbn = v_isbn;

    SELECT CONCAT('Thank you for returning the book: ', v_book_name) AS message;
END$$
DELIMITER ;

-- Usage
CALL add_return_records('RS138', 'IS135', 'Good');
```

**Advanced Task 3 — Branch Performance Report (CTAS):**
```sql
CREATE TABLE branch_reports AS
SELECT 
    b.branch_id,
    b.manager_id,
    COUNT(ist.issued_id) AS number_book_issued,
    COUNT(rs.return_id) AS number_of_book_return,
    SUM(bk.rental_price) AS total_revenue
FROM issued_status AS ist
JOIN employees AS e ON e.emp_id = ist.issued_emp_id
JOIN branch AS b ON e.branch_id = b.branch_id
LEFT JOIN return_status AS rs ON rs.issued_id = ist.issued_id
JOIN books AS bk ON ist.issued_book_isbn = bk.isbn
GROUP BY 1, 2;
```

**Advanced Task 4 — Active Members (Last 2 Years, CTAS):**
```sql
CREATE TABLE active_members AS
SELECT * FROM members m
WHERE EXISTS (
    SELECT 1 FROM issued_status ist
    WHERE ist.issued_member_id = m.member_id
      AND ist.issued_date >= NOW() - INTERVAL 2 YEAR
);
```

**Advanced Task 5 — Top 3 Employees by Books Processed:**
```sql
SELECT e.emp_name, b.*, COUNT(ist.issued_id) AS no_book_issued
FROM issued_status AS ist
JOIN employees AS e ON e.emp_id = ist.issued_emp_id
JOIN branch AS b ON e.branch_id = b.branch_id
GROUP BY 1, 2
ORDER BY no_book_issued DESC
LIMIT 3;
```

**Advanced Task 6 — Issue Book Stored Procedure (with availability check):**
```sql
DELIMITER $$
CREATE PROCEDURE issue_book(
    IN p_issued_id VARCHAR(10),
    IN p_issued_member_id VARCHAR(10),
    IN p_issued_book_isbn VARCHAR(20),
    IN p_issued_emp_id VARCHAR(10)
)
BEGIN
    DECLARE v_status VARCHAR(10);

    SELECT status
     INTO v_status FROM books
     WHERE isbn = p_issued_book_isbn;

    IF v_status = 'yes' THEN
        INSERT INTO issued_status(issued_id, issued_member_id, issued_date, issued_book_isbn, issued_emp_id)
        VALUES (p_issued_id, p_issued_member_id, CURRENT_DATE, p_issued_book_isbn, p_issued_emp_id);

        UPDATE books SET status = 'no' WHERE isbn = p_issued_book_isbn;

        SELECT CONCAT('Book records added successfully for book isbn: ', p_issued_book_isbn) AS message;
    ELSE
        SELECT 'Sorry, the requested book is currently unavailable' AS message;
    END IF;
END$$
DELIMITER ;

-- Usage
CALL issue_book('IS155', 'C108', '978-0-553-29698-2', 'E104');
```

---

## Key SQL Concepts Covered

- Table creation with primary & foreign keys
- CRUD operations (INSERT, SELECT, UPDATE, DELETE)
- Aggregate functions: `COUNT()`, `SUM()`
- Grouping and filtering: `GROUP BY`, `HAVING`
- Table joins: `INNER JOIN`, `LEFT JOIN`
- Subqueries and `EXISTS`
- `CREATE TABLE AS SELECT` (CTAS)
- Date functions: `DATEDIFF()`, `INTERVAL`, `CURRENT_DATE`
- Stored Procedures with `IN` parameters, `IF/ELSE` logic, and `DECLARE` variables
- `ALTER TABLE` to modify structure post-creation

---
THANK YOU SO MUCH 

