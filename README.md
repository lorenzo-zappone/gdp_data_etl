# GDP Data Pipeline

## Introduction

This project demonstrates a data ingestion and transformation pipeline that extracts GDP data for South American countries from the World Bank API, loads the data into a PostgreSQL database, and transforms it to produce a pivoted report of the last 5 years' GDP values for each country, presented in billions.

## Project Structure

``` php	
    ├── db
        └── init_db.sql # SQL script to initialize the database schema
    ├── scripts
        ├── extract.py # Script to extract data from the World Bank API
        ├── load.py # Script to load data into the PostgreSQL database
        └── transform.sql # SQL script to create the pivoted view
    ├── Dockerfile # Dockerfile for the application
    ├── docker-compose.yml # Docker Compose file to orchestrate services
    ├── requirements.txt # Python dependencies
    ├── run.sh # Shell script to run the entire ETL process
    └── README.md # Project documentation
```	

## Setup and Execution

### Prerequisites

- Docker
- Docker Compose
- Poetry (for dependency management and packaging)
- pgAdmin
- Astro CLI **

### Installing Prerequisites

**Docker and Docker Compose:**
Follow the installation instructions for your OS on the [Docker website](https://docs.docker.com/get-docker/).

**Poetry:**
Follow the installation instructions on the [Poetry website](https://python-poetry.org/docs/#installation).

****Optional step - Astro CLI:**
Follow the installation instructions for your OS:

- **Windows:**
    ```bash
    powershell -Command "winget install -e --id Astronomer.Astro"
    ```

- **MacOS:**
    ```bash
    brew install astro
    ```

- **Linux:**
    ```bash
    curl -sSL install.astronomer.io | sudo bash -s
    ```

## Steps to Execute

1. **Clone the repository:**

    ```bash
    git clone https://github.com/lorenzo-zappone/data-engineer-cloudwalk.git
    cd data-engineer-cloudwalk
    ```

2. **Install dependencies using Poetry:**

    ```bash
    poetry install
    ```

3. **Build and run the Docker containers:**

    ```bash
    docker-compose up --build -d
    ```

    This command will:
    - Build the Docker images.
    - Start the PostgreSQL database container.
    - Start the pgAdmin container.
    - Run the ETL process in the application container.

4. **Access pgAdmin:**

    After running the containers, pgAdmin will be available at `http://localhost:5050`.

    - **Email:** `admin@admin.com`
    - **Password:** `admin`

5. **Validate the tables in the database:**

    - Log in to pgAdmin.
    - Connect to the PostgreSQL server using the credentials provided in the `docker-compose.yml` file.
    - Verify that the `country` and `gdp` tables are created and populated.

## Optional configuration
6. **Navigate to the Airflow folder:**

    ```bash
    cd airflow
    ```

7. **Start Airflow using Astro CLI:**

    ```bash
    astro dev start
    ```

    - Open your browser and go to `http://localhost:8080` to access the Airflow web UI.

## Design Decisions and Assumptions

### Assumptions

- The World Bank API endpoint can have multiple pages. The `extract.py` script handles pagination to ensure all data is retrieved.
- The data is stored in two tables: `country` and `gdp`. The `country` table stores country metadata, while the `gdp` table stores GDP values for each year.
- The ETL process assumes that the database and tables are initially empty. The `ON CONFLICT` clause in the `INSERT` statements ensures that duplicate entries are not created.

### Design Decisions

1. **Dockerization:**
   - The project uses Docker to ensure consistent environments and dependencies across different systems.
   - Docker Compose is used to orchestrate multiple services (PostgreSQL and pgAdmin).

2. **Data Extraction:**
   - The `extract.py` script fetches GDP data from the World Bank API and saves it to a local JSON file.
   
```python
    import requests
    import json

    def extract_data():
        base_url = "https://api.worldbank.org/v2/country/"
        countries = ["ARG", "BOL", "BRA", "CHL", "COL", "ECU", "GUY", "PRY", "PER", "SUR", "URY", "VEN"]
        indicator = "NY.GDP.MKTP.CD"
        data = []

        for country in countries:
            page = 1
            while True:
                url = f"{base_url}{country}/indicator/{indicator}?format=json&page={page}&per_page=50"
                response = requests.get(url)
                json_data = response.json()

                if not json_data:
                    break

                data.extend(json_data[1])

                if len(json_data[1]) < 50:
                    break

                page += 1

        return data

if __name__ == "__main__":
    data = extract_data()
    with open("gdp_data.json", "w") as f:
        json.dump(data, f)
```

3. **Data Loading:**
   - The `load.py` script reads the JSON data and inserts it into the PostgreSQL database. It handles potential conflicts by using `ON CONFLICT` clauses.

```python
import json
import psycopg2

def load_data(filename='gdp_data.json'):
    with open(filename, 'r', encoding='utf-8') as f:
        return json.load(f)

def connect_db():
    try:
        conn = psycopg2.connect(
            dbname='gdp_data',
            user='cloudwalk',
            password='EzOiDSqfrdy5cbkXhr2LQHN6eB1SzE3O',
            host='dpg-cp4lc5779t8c73ei01pg-a.oregon-postgres.render.com'
        )
        return conn
    except Exception as e:
        print(f"Error connecting to the database: {e}")
        raise

def create_tables(conn):
    try:
        with conn.cursor() as cur:
            # Open and read the SQL script using Python's built-in file handling
            with open('db/init_db.sql', 'r', encoding='utf-8') as sql_file:
                sql_script = sql_file.read()

            # Execute the SQL script
            cur.execute(sql_script)
            
        conn.commit()
    except Exception as e:
        print(f"Error creating tables: {e}")
        raise

def insert_data(conn, data):
    try:
        with conn.cursor() as cur:
            for entry in data:
                country_id = entry['country']['id']
                country_name = entry['country']['value']
                iso3_code = entry['countryiso3code']
                year = entry['date']
                value = entry['value']

                cur.execute("""
                    INSERT INTO country (id, name, iso3_code)
                    VALUES (%s, %s, %s)
                    ON CONFLICT (id) DO NOTHING;
                """, (country_id, country_name, iso3_code))

                cur.execute("""
                    INSERT INTO gdp (country_id, year, value)
                    VALUES (%s, %s, %s)
                    ON CONFLICT (country_id, year) DO NOTHING;
                """, (country_id, year, value))
                    
        conn.commit()
    except Exception as e:
        print(f"Error inserting data: {e}")
        raise

if __name__ == '__main__':
    data = load_data()
    conn = connect_db()
    create_tables(conn)
    insert_data(conn, data)
    conn.close()
```

4. **Data Transformation:**
   - The `transform.sql` script creates a pivoted view `pivoted_gdp` to facilitate reporting. This view presents the last 5 years' GDP values in billions.

```sql
CREATE VIEW pivoted_gdp AS
SELECT
    country.id,
    country.name,
    country.iso3_code,
    ROUND(MAX(CASE WHEN gdp.year = 2019 THEN gdp.value / 1e9 END), 2) AS "2019",
    ROUND(MAX(CASE WHEN gdp.year = 2020 THEN gdp.value / 1e9 END), 2) AS "2020",
    ROUND(MAX(CASE WHEN gdp.year = 2021 THEN gdp.value / 1e9 END), 2) AS "2021",
    ROUND(MAX(CASE WHEN gdp.year = 2022 THEN gdp.value / 1e9 END), 2) AS "2022",
    ROUND(MAX(CASE WHEN gdp.year = 2023 THEN gdp.value / 1e9 END), 2) AS "2023"
FROM
    country
JOIN
    gdp ON country.id = gdp.country_id
GROUP BY
    1, 2, 3
ORDER BY
    2 ASC;
```

5. **Automation:**
   - The `run.sh` script automates the ETL process, including data extraction, loading, and transformation when the docker container is created.

## Running the ETL Process Manually

If you need to run the ETL process manually, you can do so by executing the scripts in the following order:

1. **Extract Data:**

    ```bash
    docker-compose exec app python scripts/extract.py
    ```

2. **Load Data:**

    ```bash
    docker-compose exec app python scripts/load.py
    ```

3. **Transform Data:**

    ```bash
    docker-compose exec db psql -U postgres -d gdp_data -f /files/transform.sql
    ```

## Conclusion

This project provides a robust and scalable solution for ingesting and transforming GDP data. The use of Docker ensures that the solution can be easily deployed and maintained. The documentation and clear structure make it easy to understand and extend.

If you have any questions or need further assistance, please feel free to contact zappone500@gmail.com.
