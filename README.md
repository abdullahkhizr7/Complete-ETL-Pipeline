End-to-End ETL Pipeline for Order Processing

This project demonstrates a complete, end-to-end ETL (Extract, Transform, Load) pipeline built in Python. The pipeline extracts nested JSON order data from an AWS S3 bucket, transforms it into a relational star schema, and loads it into a MySQL data warehouse.

The project is divided into two main parts:

1.  Initial Data Load: Populates the database with an initial batch of data.
2.  Incremental Update/Load: Processes a second file to update existing records and insert new ones, demonstrating a common data warehousing pattern for handling new or changed data.

---

Key Concepts and Features

This project serves as a practical example of several important data engineering and data warehousing concepts.

Star Schema & Surrogate Keys

The pipeline is designed to load data into a star schema, which consists of a central fact table (orders) surrounded by dimension tables (products, customers).

* Dimension Tables: products and customers store descriptive attributes (like product name, category, customer name, email).
* Fact Table: orders stores quantitative measures (like total_amount, quantity) and, most importantly, surrogate keys that link to the dimension tables.
* Surrogate Keys (SKs): Instead of using natural business keys (like product_id or customer_id) as primary keys, the dimension tables are designed to use database-generated surrogate keys (e.g., product_sk, customer_sk). The initial load script logic is built around this concept. It first loads the dimensions, then re-reads them to fetch the generated SKs, which are then used to populate the orders fact table. This is a best practice that improves join performance and decouples the warehouse from source system key changes.

Slowly Changing Dimensions (SCD) - Type 1

The second script (Updating and Adding Data) implements a Slowly Changing Dimension (SCD) Type 1 strategy. This method is used to manage changes in dimension data.

* How it works: When new data arrives (from orders_ETL_incremental.json), it's first loaded into temporary staging tables (products_sk, customers_sk).
* Update (SCD Type 1): A SQL UPDATE ... INNER JOIN query is executed. If a record from the staging table matches an existing record in the main dimension table (e.g., same product_id), the existing record is overwritten with the new information (e.g., a new price or name). This approach does not keep a history of old values.
* Insert (New Records): A SQL INSERT INTO ... SELECT ... WHERE NOT IN query is executed to add any brand-new dimension records (e.g., new customers or products) that don't already exist in the main tables.

SQLAlchemy: ORM and Raw SQL

The project uses SQLAlchemy as the database interaction layer, demonstrating its flexibility:

* Engine Creation: create_engine is used to establish a connection to the MySQL database.
* Pandas Integration: pandas.to_sql is used for simple, efficient bulk-loading of DataFrames into the database (e.g., loading staging tables).
* Raw SQL Execution: For the more complex SCD logic, SQLAlchemy's text function and engine.connect() are used to execute raw SQL UPDATE and INSERT statements. This shows how to handle operations that are more complex than a simple table drop/replace.

---

Technology Stack

* Python 3
* Pandas: For all data manipulation and transformation tasks.
* Boto3: The AWS SDK for Python, used to extract JSON files from S3.
* SQLAlchemy: For connecting to and interacting with the MySQL database.
* PyMySQL: The MySQL driver used by SQLAlchemy.
* AWS S3: The data source (Extract).
* MySQL: The target data warehouse (Load).

---

How the Pipeline Works

1. Initial Load

1.  Extract: The extract() function uses boto3 to connect to AWS S3 and pull the orders_ETL.json file. The JSON data is loaded into memory.
2.  Transform: The tranform() function flattens the nested JSON. It iterates through the orders and their nested products/customers to create three separate lists of dictionaries, which are then converted into Pandas DataFrames:
    * df_orders (fact data)
    * df_products_dim (product dimension)
    * df_customers_dim (customer dimension)
    * Dimension data is deduplicated using drop_duplicates(..., keep='last') to ensure only the most recent version of each entity is kept.
3.  Load: The load() function:
    * Appends the df_products_dim and df_customers_dim DataFrames to the products and customers tables in MySQL.
    * Fetches Surrogate Keys: It then reads the full products and customers tables back from MySQL (which now include the database-generated surrogate keys).
    * Merges SKs: It merges these surrogate keys into the df_orders fact DataFrame based on the natural keys (product_id, customer_id).
    * Loads Fact Table: The final df_orders_final DataFrame, now containing the correct surrogate keys, is loaded into the orders table using if_exists='replace'.

2. Incremental Update (SCD Type 1)

1.  Extract & Transform: The process is identical to the initial load, but it uses the orders_ETL_incremental.json file.
2.  Load: This load() function is different:
    * It loads the new/updated dimension data into temporary staging tables (products_sk, customers_sk) using if_exists='replace'.
    * It executes a raw SQL UPDATE query to find matching records and overwrite their attributes (SCD Type 1).
    * It executes a raw SQL INSERT query to add any new products or customers that are not already in the dimension tables.
    * (The fact-loading portion is commented out but would follow the same logic as the initial load: fetch SKs, merge, and append to the fact table).

---

How to Run

1. Prerequisites

* A running MySQL server.
* An AWS S3 bucket containing the orders_ETL.json and orders_ETL_incremental.json files.

2. Install Dependencies

    pip install pandas sqlalchemy pymysql boto3

3. Database Setup (Critical)

Before running the first script, you MUST create the products, customers, and orders tables in your MySQL database. They must include auto-incrementing surrogate primary keys.

*Example SQL:*

    CREATE TABLE products (
        product_sk INT AUTO_INCREMENT PRIMARY KEY,
        product_id INT,
        name VARCHAR(255),
        category VARCHAR(255),
        price DECIMAL(10, 2),
        UNIQUE(product_id)
    );

    CREATE TABLE customers (
        customer_sk INT AUTO_INCREMENT PRIMARY KEY,
        customer_id INT,
        name VARCHAR(255),
        email VARCHAR(255),
        address VARCHAR(255),
        UNIQUE(customer_id)
    );

    CREATE TABLE orders (
        order_id INT,
        order_date DATE,
        total_amount DECIMAL(10, 2),
        product_sk INT,
        customer_sk INT,
        quantity INT,
        FOREIGN KEY (product_sk) REFERENCES products(product_sk),
        FOREIGN KEY (customer_sk) REFERENCES customers(customer_sk)
    );

4. Configure Credentials

Update the following in your Python script(s):

* AWS: Update the boto3.client call in the extract() function with your aws_access_key_id and aws_secret_access_key.
* MySQL: Update the create_engine string in the main() function with your MySQL username, password, host, port, and database name (e.g., mysql+pymysql://user:password@host:port/database).

5. Run the Pipeline

1.  Execute the first Python cell/script (Initial Load) to populate your database.
2.  Execute the second Python cell/script (Updating and Adding Data) to process the incremental file and perform the SCD Type 1 update.
