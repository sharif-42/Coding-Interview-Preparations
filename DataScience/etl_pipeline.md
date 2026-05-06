# ETL Pipeline
Data rarely lives in the format you need it. Your production database stores normalized rows optimized for transactions. Your analytics team needs denormalized tables optimized for queries. Your machine learning models need clean, feature-engineered datasets. Your data warehouse needs aggregated historical data.

Getting data from where it is to where it needs to be, in the format it needs to be in, is the job of **ETL: Extract, Transform, Load**. ETL pipelines are the plumbing of data infrastructure. They run silently in the background, moving terabytes of data every night, and only get attention when something breaks.

Understanding ETL is essential for system design. Almost every data architecture involves moving data between systems, and the patterns you learn here apply whether you are using batch processing, streaming, or hybrid approaches.

Before deep drive into it lets know the types of data around us.

## Types of data
1. **Structured data**: Structured data is easy to search and organize. Data is entered following a rigid structure, like a spreadsheet where there are set columns. Each column takes values of a certain type, like text, data, or decimal. It makes it easy to form relations, hence it's organized in what is called a relational database. About 20% of the data is structured. SQL, which stands for Structured Query Language, is used to query such data.
- Relational Database

2. **Semi-Structured data**: Semi-structured data resembles structured data, but allows more freedom. It's therefore relatively easy to organize, and pretty structured, but allows more flexibility. It also has different types and can be grouped to form relations, although this is not as straightforwards as with structured data - you have to pay for that flexibility at some point. 
- Stored in NoSQL databases (as opposed to SQL)
- JSON, XML or YAML file formats.

3. **Unstructured data**: Unstructured data is data that does not follow a model and can't be contained in a rows and columns format. This makes it difficult to search and organize.
- Schemaless
- More like files
- Usually text, sound, pictures or videos.

## What is ETL?
ETL stands for **Extract, Transform, Load**. It describes the process of moving data from source systems to destination systems while changing its format.
![ETL Pipeline](/DataScience/images/etl_pipeline_1.png)

### The Three Phases

| Phase     | Purpose                              | Example                                                     |
|----------|--------------------------------------|-------------------------------------------------------------|
| Extract  | Read data from source systems        | Query production database, call APIs, read files            |
| Transform| Convert data to desired format       | Clean, validate, aggregate, join, denormalize               |
| Load     | Write data to destination system     | Insert into warehouse, update tables   

### Why ETL Matters
Without ETL, every system would need to:
- Understand every source system's format
- Handle all data quality issues itself
- Maintain connections to all data sources
- Compete with production systems for resources
ETL centralizes these concerns:
![ETL Pipeline](/DataScience/images/etl_pipeline_2.png)

## Extract Phase
Extraction is the process of retrieving data from heterogeneous source systems such as databases, APIs, or files, while ensuring minimal impact on the source system. The Extract phase reads data from source systems. This sounds simple but involves significant complexity. 

| Source Type          | Examples                              | Challenges                                      |
|---------------------|----------------------------------------|------------------------------------------------|
| Relational databases| MySQL, PostgreSQL, Oracle              | Connection limits, query impact on production  |
| NoSQL databases     | MongoDB, Cassandra, DynamoDB           | No standard query language, denormalized data  |
| APIs                | REST, GraphQL, SOAP                    | Rate limits, pagination, authentication        |
| Files               | CSV, JSON, Parquet, logs               | Schema variations, encoding issues             |
| Message queues      | Kafka, RabbitMQ                        | Ordering, exactly-once processing              |
| SaaS platforms      | Salesforce, Stripe, HubSpot            | API limits, data access restrictions           |

**Full Extract**: 
- Read all rows every time. 
- Simple, always consistent
- Slow, wastes resources, strains source

**Incremental Extract**: 
- Read only NEW/CHANGED rows since last run
- Fast, minimal source impact
- Requires change tracking, can miss deletes


## Transform Phase
Transformation is the process of cleaning, validating, enriching, and converting raw data into a structured and consistent format that aligns with business rules and target schema requirements.

The Transform phase converts raw data into the format needed by destination systems. This is where most of the business logic lives.

### Common Transformations
![ETL Pipeline](/DataScience/images/etl_pipeline_3.png)

### Transformation Types
| Type            | Description                         | Example                                              |
|-----------------|-------------------------------------|------------------------------------------------------|
| Cleaning        | Fix data quality issues             | Remove nulls, fix encoding, trim whitespace          |
| Validation      | Ensure data meets requirements      | Check required fields, validate formats              |
| Standardization | Make data consistent                | Convert dates to UTC, normalize phone numbers        |
| Enrichment      | Add derived data                    | Calculate age from birthdate, add geo data from IP   |
| Aggregation     | Summarize data                      | Sum daily sales, count monthly users                 |
| Joining         | Combine data sources                | Join orders with customers                           |
| Filtering       | Remove unwanted data                | Exclude test accounts, filter by date range          |
| Deduplication   | Remove duplicates                   | Dedupe by user email                                 |
| Denormalization | Flatten for analytics               | Embed customer data in order records                 |

### Data Quality Checks
Transform is where you enforce data quality:
![ETL Pipeline](/DataScience/images/etl_pipeline_4.png)

**Common quality checks:**
- Null checks: Critical fields must have values
- Type checks: Dates are dates, numbers are numbers
- Range checks: Values within expected bounds
- Format checks: Email, phone, postal code formats
- Referential checks: Foreign keys exist in parent tables
- Uniqueness checks: Primary keys are unique
- Business rules: Order total matches line items sum

## Load Phase
The Load phase writes transformed data to destination systems. The strategy depends on the destination type and use case.

### Load Strategies

![ETL Pipeline](/DataScience/images/etl_pipeline_5.png)

| Strategy           | When to Use                           | How It Works                                      |
|--------------------|----------------------------------------|--------------------------------------------------|
| Full load          | Small tables, complete refresh needed | Truncate and reload                              |
| Incremental append | Event/transaction tables              | Insert new rows only                             |
| Upsert (merge)     | Dimension tables                      | Insert new, update existing                      |
| SCD Type 2         | Need historical tracking              | New row for each change, track valid dates       |

### Bulk Loading Techniques
For large data volumes, row-by-row inserts are too slow:
| Technique         | Description                          | Speed                         |
|------------------|--------------------------------------|-------------------------------|
| Batch INSERT     | Multi-row INSERT statements          | 10x faster than single        |
| COPY command     | Load from files (Postgres, Redshift) | 100x faster                   |
| Bulk loader      | Native database utilities            | Fastest                       |
| Parallel loading | Multiple concurrent loaders          | Scales with connections       |

### Staging Tables
A common pattern is to load into staging tables first:
![ETL Pipeline](/DataScience/images/etl_pipeline_6.png)

Benefits:
- Validate before affecting target
- Atomic swap (rename tables)
- Rollback if issues found
- Target table always consistent


## ETL vs ELT
Modern data platforms have shifted from ETL to ELT.

**ETL**: Transform Before Load
![ETL Pipeline](/DataScience/images/etl_pipeline_7.png)
Transformations happen on ETL servers before loading.

**ELT**: Transform After Load
![ETL Pipeline](/DataScience/images/etl_pipeline_8.png)
Raw data is loaded first, transformations happen in the warehouse. With ELT

- Load raw data quickly (minimal processing)
- Transform using warehouse SQL
- Re-transform anytime without re-extracting
- Keep raw data for audit and new use cases

### Comparison
| Aspect              | ETL                                   | ELT                                      |
|---------------------|----------------------------------------|-------------------------------------------|
| Transform location  | ETL server                             | Data warehouse                            |
| Compute cost        | ETL infrastructure                     | Warehouse compute                         |
| Raw data            | Not preserved                          | Preserved in warehouse                    |
| Flexibility         | Changes require ETL updates            | Transform with SQL anytime                |
| Best for            | Structured, well-defined transformations| Exploration, evolving requirements        |
| Tools               | Informatica, Talend, SSIS              | dbt, Dataform, warehouse SQL              |


### Why ELT is Gaining Popularity
Modern cloud warehouses (Snowflake, BigQuery, Redshift) have:
- Virtually unlimited compute
- Columnar storage for fast analytics
- SQL-based transformations
- Lower cost than maintaining ETL servers

## ETL Pipeline Patterns
### The Medallion Architecture
A popular pattern for organizing data in lakes and warehouses:
![ETL Pipeline](/DataScience/images/etl_pipeline_9.png)

| Layer             | Purpose                          | Users                         |
|------------------|----------------------------------|-------------------------------|
| Bronze (Raw)     | Exact copy of source             | Data engineers, debugging     |
| Silver (Cleaned) | Validated, standardized          | Data scientists, analysts     |
| Gold (Aggregated)| Business metrics, KPIs           | Dashboards, reports           |

### Change Data Capture Pipeline
**For real-time or near-real-time data sync**: CDC captures every insert, update, and delete from the source database's transaction log, enabling near-real-time replication.
![ETL Pipeline](/DataScience/images/etl_pipeline_10.png)

### Dependency DAG
**Complex ETL involves many interdependent jobs**: Orchestration tools (Airflow, Dagster, Prefect) manage these dependencies.

![ETL Pipeline](/DataScience/images/etl_pipeline_11.png)


```python
# Connect postgresql to pandas dataframe

# Function to extract table to a pandas DataFrame
def extract_table_to_pandas(tablename, db_engine):
    query = "SELECT * FROM {}".format(tablename)
    return pd.read_sql(query, db_engine)

# Connect to the database using the connection URI
connection_uri = "postgresql://repl:password@localhost:5432/pagila" 
db_engine = sqlalchemy.create_engine(connection_uri)

# Extract the film table into a pandas DataFrame
extract_table_to_pandas("film", db_engine)

# Extract the customer table into a pandas DataFrame
extract_table_to_pandas("customer", db_engine)
```