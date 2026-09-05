# BinDB Library

Author: tuonux \<tuonux0@gmail.com\>

BinDB it's a library for Grey Hack Game that allow you to create and manage a password-protected binary database inaccessible to third parties and without need parsers or something like that.

## How to use it

At the top of everything to your code import libbindb.src with the "import_code" method.

    import_code("/absolute/path/of/libbindb.src")

After that, create a connection with your db with the following example method:

    myDb = BinDB.connect("dbname", "dbpassword", ["table1", "table2"], "/home/tuonux")

## Availables methods

**_Keep in mind: The index of the rows starts form 1 and not 0_**

### BinDB.connect(dbname, dbpassword, tablesArray, dbDirectory)

Instantiate the connection with your binary database

Example Usage:

    mrRobotDb = BinDB.connect("mrRobot", "mypassword", ["users", "mails", "banks"], "/home/<user>")

### BinDB.insert(table, data)

Push the new data in your table

Example Usage:

    mrRobotDb.insert("users", {"name": "Elliot", "surname": "Alderson"})
    mrRobotDb.insert("users", {"name": "Tyrell", "surname": "Wellick"})
    mrRobotDb.insert("users", {"name": "Angela", "surname": "Moss"})
    mrRobotDb.insert("users", {"name": "Joanna", "surname": "Olofsson"})
    mrRobotDb.insert("users", {"name": "Gideon", "surname": "Goddard"})

### BinDB.fetch(table)

Fetch all the rows of your table

Example Usage:

    for user in mrRobotDb.fetch("users")
         print(user.name)
    end for

### BinDB.fetchOne(table, id)

Fetch a row by the index

Example Usage:

    userElliot = mrRobotDb.fetchOne("users", 1)

### BinDB.fetchBy(table, key, value)

Fetch a row by a combination of key -> value

Example Usage:

    userElliot = mrRobotDb.fetchBy("users", "name", "Elliot")

### BinDB.update(table, id, data)

Update a row by the index

Example Usage:

    mrRobotDb.update("users", 1, {"name": "Mr.", "surname": "Robot"})

### BinDB.delete(table, id)

Delete a row by the index

Example Usage:

    mrRobotDb.delete("users", 5)

### BinDB.read()

Read binary buffer ( used in rare case )

Example Usage:

    mrRobotDb.read()

### BinDB.write()

Update binary database buffer

Example Usage:

    mrRobotDb.insert("users", {"name": "Elliot", "surname": "Alderson"})
    mrRobotDb.write()

### BinDB.wipe()

Clear and delete the database

Example Usage:

    mrRobotDb.wipe()

### BinDB.printTable(table, labels)

Utility function that print your table with formatted columns

Example Usage:

    mrRobotDb.printTable("users", {"name": "Name", "surname": "Surname"})

### BinDB.count(table)

Count the rows of a table

Example Usage:

    mrRobotDb.count("users")

### BinDB.join(tableA, tableB, keyA, keyB, type)

Join two tables on a key (nested-loop join). `keyB` defaults to `keyA`, `type` defaults to `"inner"` (only matching rows); pass `"left"` to also keep unmatched rows from `tableA`. Matching rows are merged into a single map whose keys are prefixed with the table name to avoid collisions.

Example Usage:

    mrRobotDb.join("users", "mails", "id", "userId")

    // returns something like:
    // [{"users.id": 1, "users.name": "Elliot", "mails.userId": 1, "mails.address": "elliot@ecorp.com"}]

### BinDB.query(table)

Start a SQL-like fluent query on a table. Returns a `BinDBQuery` you chain conditions/ordering/pagination onto, then run with a terminal method.

**Where clauses**

- `.where(key, value)` / `.where(key, operator, value)` — AND condition. Supported operators: `=`, `!=`, `>`, `<`, `>=`, `<=`, `like`, `in`, `between`
- `.orWhere(key, [operator,] value)` — OR condition
- `.whereLike(key, pattern)` — SQL-style `LIKE`, `%` as wildcard (`"%foo"`, `"foo%"`, `"%foo%"`), case-insensitive
- `.whereIn(key, values)` — matches if the key's value is in the given list
- `.whereBetween(key, min, max)` — inclusive range

Conditions are evaluated left to right in the order they were added (no parenthesis grouping), same as chaining `where`/`orWhere` in most query builders.

**Ordering & pagination**

- `.orderBy(key, direction)` — `direction` is `"asc"` (default) or `"desc"`
- `.limit(n)` / `.offset(n)`

**Terminal methods**

- `.get()` / `.all()` — run the query, return matching rows
- `.first()` — run the query, return the first matching row (or `null`)
- `.count()` — number of matching rows (ignores `orderBy`/`limit`/`offset`)
- `.sum(key)` / `.avg(key)` / `.min(key)` / `.max(key)` — aggregate over matching rows
- `.distinct(key)` — unique values of a key over matching rows
- `.update(data)` — merges `data` into every matching row (unlike `BinDB.update`, it does not replace the whole row). Returns the number of affected rows
- `.delete()` — removes every matching row. Returns the number of affected rows

Example Usage:

    adults = mrRobotDb.query("users").where("age", ">=", 18).orderBy("name").limit(10).get()

    elliot = mrRobotDb.query("users").whereLike("name", "%elliot%").first()

    affected = mrRobotDb.query("users").where("surname", "Alderson").update({"surname": "Robot"})

    removed = mrRobotDb.query("users").where("name", "Gideon").delete()

    total = mrRobotDb.query("users").count()

### Methods chain

For your convenience you can chain the methods if you want

    mrRobotDb.insert("users", {"name": "Terry", "surname": "Colby"}).write()

## What if another user attempt to launch the binary database?

If someone tries to launch your database executable, an information message will be shown

    tuonux@PC:~$ /home/tuonux/mrRobot.db

    This is a binary database generated by BinDB Library
    Info: https://github.com/tuonux/gh-bindb

    tuonux@PC:~$

If someone tries to launch your database executable by passing a parameter and the password is incorrect, an error message will be shown

    tuonux@PC:~$ /home/tuonux/mrRobot.db pass
    Permission denied
    tuonux@PC:~$

## Protect the source code of your program

As long as you have your data protected in a binary database you are safe, but don't forget that as with any interpreted language, the source code of the code where you include the BinDB library could be visible to other people and they could find out your password.

Upload only the binary of your program or make sure you have the right permissions set on your file where the source code for your program resides.

## Next Goals:

- Terminal GUI to explore your database in your terminal and perform queries directly from it
- Raw SQL-like string queries (e.g. `db.sql("SELECT * FROM users WHERE age > 18")`)
- Parenthesis grouping for where/orWhere conditions

## License

    MIT License

    Copyright (c) 2023 tuonux

    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:

    The above copyright notice and this permission notice shall be included in all
    copies or substantial portions of the Software.

    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
    SOFTWARE.
