Using Psycopg / Attention: no info on setup and connect to postgres. 
/used tricky internet recommendations.
No one has completed this Reading lately, be the first!
Using Psycopg
Just as you used the pg library in JavaScript to connect to PostgreSQL, you will use a package in Python to connect to and interact with the RDBMS. Unlike JavaScript, there are four actively maintained Python packages that provide connectivity to PostgreSQL. This article presents Psycopg (pronounced like "Psycho P.G.") which is the default used by SQLAlchemy, the ORM that you will use in later lessons. By the end of this article, you should be able to:

Connect to a PostgreSQL RDBMS using Psycopg
Open a "cursor" to perform data operations
Use results performed from executing a SELECT statement on existing database entities
Use parameterized SQL statements to select, insert, update, and delete data
Specify what type Psycopg will convert the following PostgreSQL types into:
NULL
bool
double
integer
varchar
text
date
Use the with keyword to clean up connections and database cursors
Install Psycopg
To be able to connect to a PostgreSQL RDBMS (Postgres) using Psycopg, you must have Postgres installed. If you are unsure if it is installed verify in the terminal:

postgres --version
If this command does not respond with a version greater than 12, follow the steps necessary to install or upgrade Postgres for your environment.

You also need a program called pg_config installed. This is usually installed as a part of the Postgres installation, but sometimes you need to manually add it to PATH. You can verify that it is there with:

pg_config --version
If the response is not the version of Postgres matching the result of the first command, you will need to locate pg_config and set the path. It can usually be found at /usr/lib/postgresql/X.Y/bin/, but it's possible that the installation can be in a different location. Add it to PATH:

export PATH=/usr/lib/postgresql/X.Y/bin/:$PATH
You should now be ready to install the package with pipenv as you would any other. We'll use the binary version to avoid a few potential pitfalls.

pipenv install psycopg2-binary
Don't forget the 2!

Setting up your database
Use the following SQL statements to set up your database.

-- This is really bad security and should not be done in real-world
-- application programming.
CREATE USER psycopg_test_user WITH CREATEDB PASSWORD 'password';
-- Must be run by itself
CREATE DATABASE psycopg_test_db WITH OWNER psycopg_test_user;
Now, connect to the psycopg_test_db database as the psycopg_test_user user with the password "password".

You will know that you are in the correct database when the command prompt has the correct name: psycopg_test_db=#

Set up the rest of the database you will use for this exercise.

CREATE TABLE owners (
  id SERIAL PRIMARY KEY,
  first_name VARCHAR(255) NOT NULL,
  last_name VARCHAR(255) NOT NULL,
  email VARCHAR(255) NOT NULL
);

-- Make and model should have their own tables
-- Simplified for now
CREATE TABLE cars (
  id SERIAL PRIMARY KEY,
  manu_year INTEGER NOT NULL,
  make VARCHAR(255),
  model VARCHAR(255),
  owner_id INTEGER NOT NULL,
  FOREIGN KEY (owner_id) REFERENCES owners(id)
);

INSERT INTO owners (first_name, last_name, email)
VALUES
('Tim', 'Petrol', 'rotary@fast.com'),
('Ryan', 'Runner', '10sec@jdm.com'),
('Tia', 'Petrol', 'typer@wtec.com');

INSERT INTO cars (manu_year, make, model, owner_id)
VALUES
(1993, 'Mazda', 'Rx7', 1),
(1995, 'Mitsubishi', 'Eclipse', 2),
(1994, 'Acura', 'Integra', 3);
Connect to a PostgreSQL RDBMS using Psycopg
From here on out, you'll use Psycopg to manipulate the database. First, create a file named psycopg_demo.py and paste the code below to connect to the database. Then run your file with pipenv run python psycopg_demo.py.

import psycopg2

CONNECTION_PARAMETERS = {
                          'dbname': 'psycopg_test_db',
                          'user': 'psycopg_test_user',
                          'password': 'password',
}

with psycopg2.connect(**CONNECTION_PARAMETERS) as conn:
    print(conn.get_dsn_parameters())

    # Output: {'user': 'psycopg_test_user', 'dbname': 'psycopg_test_db', ...}
Open a "cursor" to perform data operations
After you've connected, you need to create a cursor. Add another with statement to do so.

with psycopg2.connect(**CONNECTION_PARAMETERS) as conn:
    with conn.cursor() as curs:
        curs.execute("SELECT USER;")
        result = curs.fetchone()
        print(result) # 'psycopg_test_user'
Use the dir() method on the curs object to explore the methods and properties available. You can also find detailed information in the documentation for the cursor class.

Use the with keyword to clean up connections and database cursors
By using the with keyword, you have simplified the creation and cleanup of your database connection and cursor. If you look for resources on the internet, you'll probably also see it done this way:

# Reference only.  The method above is better.
import psycopg2

CONNECTION_PARAMETERS = {
                          'dbname': 'psycopg_test_db',
                          'user': 'psycopg_test_user',
                          'password': 'password',
}

# Connect to the database
conn = psycopg2.connect(**CONNECTION_PARAMETERS)

# Create a cursor
curs = conn.cursor()

# Do stuff here

# Commit the changes to the database
conn.commit()

# Close the connection
cur.close()
conn.close()

Use results performed from executing a SELECT statement on existing database
The execute method is used to run any of the SQL commands that you are familiar with. Keep in mind that these do vary from implementation to implementation. For example, MySQL requires () when you execute a command to SELECT USER().

Cursor objects are iterable. Instead of using fetchone(), you can grab all of the results obtained by the cursor with fetchall(). You can then iterate over them.

with psycopg2.connect(**CONNECTION_PARAMETERS) as conn:
    with conn.cursor() as curs:
        curs.execute('SELECT manu_year, make, model FROM cars;')
        cars = curs.fetchall()
        for car in cars:
            print(car) # (1993, 'Mazda', 'Rx7')
        
Each result is a tuple containing the information in each row returned.

Use parameterized SQL statements to select, insert, update, and delete data
To start, begin converting your work into a more useful program by separating concerns into a few different functions. Begin by moving the code to return all cars into a function.

def print_all_cars():
    with psycopg2.connect(**CONNECTION_PARAMETERS) as conn:
        with conn.cursor() as curs:
            curs.execute('SELECT manu_year, make, model, owner_id FROM cars;')
            cars = curs.fetchall()
            for car in cars:
                print(car)  


print_all_cars()
# Output:
# (1993, 'Mazda', 'Rx7', 1)
# ...additional cars
The cursor will let you use parameterized SQL statements to execute your commands. Next, add a function that will return all of the cars owned by a given user.

def get_owners_cars(owner_id):
    """
    Fetch and return all cars in the cars table
    :param owner_id: <int> the id of the owner who's cars to return
    :return: <list> the results of the query
    """
    with psycopg2.connect(**CONNECTION_PARAMETERS) as conn:
        with conn.cursor() as curs:
            curs.execute("""
                         SELECT manu_year, make, model FROM cars
                         WHERE owner_id = %(owner_id)s
                         """,
                         {'owner_id': owner_id})
            results = curs.fetchall()
            return results

print(get_owners_cars(1)) # [(1993, 'Mazda', 'Rx7')]
You also need a way to add a new car into the car table. You can do that by combining the INSERT command that you are familiar with and more parameters.

def add_new_car(manu_year, make, model, owner_id):
    """
    Add the given car to the database
    :param manu_year: <int> the year the car was made
    :param make: <string> the manufacturer of the car
    :param model: <string> the model of the car
    :param owner_id: <int> the id number of the owner
    """
    with psycopg2.connect(**CONNECTION_PARAMETERS) as conn:
        with conn.cursor() as curs:
            # curs.execute(f'INSERT INTO {table}{columns} VALUES{values};')
            curs.execute("""
                         INSERT INTO cars (manu_year, make, model, owner_id)
                         VALUES (%(manu_year)s, %(make)s,
                         %(model)s, %(owner_id)s)
                         """,
                         {'manu_year': manu_year,
                          'make': make,
                          'model': model,
                          'owner_id': owner_id})


add_new_car(2000, 'Ford', 'Lightning', 2)

add_new_car(1994, 'Toyota', 'Supra', 2)    

print_all_cars()
# Output:
# ...additional cars
# (2000, 'Ford', 'Lightning', 2)
# (1994, 'Toyota', 'Supra', 2)
Next, construct a function that can update the owner of a car by id number for each record.

def change_car_owner(car_id, new_owner_id):
    """
    Update the owner of a car, both by record id
    :param car_id: <int> the id of the car to change
    :param new_owner_id: <int> the owner_id to give ownership to
    """
    with psycopg2.connect(**CONNECTION_PARAMETERS) as conn:
        with conn.cursor() as curs:
            curs.execute("""
                         UPDATE cars SET owner_id = %(new_owner_id)s
                         WHERE id = %(car_id)s
                         """,
                         {'car_id': car_id,
                          'new_owner_id': new_owner_id})


change_car_owner(5, 1)

print_all_cars()
# Output:
# ...additional cars
# (1994, 'Toyota', 'Supra', 1) <- Owner is now 1
Finally, build a function to delete the record of a car.

def delete_car(car_id):
    """
    Delete the record for a car given an id for that car
    :param car_id: <int> the id of the car record to remove
    """
    with psycopg2.connect(**CONNECTION_PARAMETERS) as conn:
        with conn.cursor() as curs:
            curs.execute("""
                         DELETE FROM cars WHERE id = %(car_id)s
                         """,
                         {'car_id': car_id})


delete_car(2)

print_all_cars()
# Output:
# (1993, 'Mazda', 'Rx7', 1)
# (1994, 'Acura', 'Integra', 3)
# (2000, 'Ford', 'Lightning', 2)
# (1994, 'Toyota', 'Supra', 1)
Specify what type Psycopg will convert PostgreSQL types into
As you may remember, Python and PostgreSQL do not have a perfect, 1-to-1 match between data types. As you convert back and forth from one to the other, conversions will occur as according to the following table.

| PostgreSQL | Python | |---------|----------| | NULL | None | | bool | bool | | double | float | | integer | long | | varchar | str | | text | unicode | | date | date | Full Conversion Table

What you've learned
In this article, you learned how to:

Connect to a PostgreSQL RDBMS using Psycopg
Open a "cursor" to perform data operations
Use results performed from executing a SELECT statement on existing database entities
Use parameterized SQL statements to select, insert, update, and delete data
Specify what type Psycopg will convert the following PostgreSQL types into:
NULL
bool
double
integer
varchar
text
date
Use the with keyword to clean up connections and database cursors