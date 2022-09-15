---
layout: post
title:  "Using Sqlite to Share Data Between Parallel Test Workers"
date:   2021-08-02 10:37:37 -0400
categories: testing
---

## A Crash Course In Test Collisions
![What happens when you drink and write automated tests.](/assets/images/crash.jpg)

When writing automated acceptance tests, one common obstacle that pops up is data collisions. This is essentially where two tests try to access the same data domain in such a way that the tests actually effect each other by effecting the product state. This is bad. Each test should be able to run as though in perfect isolation. This problem can be compounded when trying to execute tests in parallel, especially if the product itself relies on user state. I'll explain that a bit further in a minute.

First, though, I just want to say that there are manifold ways of avoiding this and it is nothing to be feared. It will probably be best for me outline a more specific situation to give a clearer context, but use your imagination here as this could apply to you and your product. 

## The Background
Recently, I was tasked with building an automated end to end test suite from the ground up for an e-commerce product that requires user authentication for all aspects of the site. That is to say, there is no way to engage with the site generically, one must have an account and be logged in. This poses a few potential problems for testing, one being that there must be some way to securely provide credentials during the test, and another being the potential for two parallel tests to effect the same user's state, causing a collision.

Now this is going to get a little reductive because, like I said, there are a number of ways to solve these issues, but I'll just be putting forth my approach. Fortunately we have a way to mitigate the credential issue so I won't talk about that any further. The main issue I'll address is collision avoidance. To lay it out, our core tests involve the user essentially taking the happy path from logging in all the way to checking out. This includes browsing for items, adding to cart, selecting shipping options, etc. For the sake of data cleanliness, at the start of each of these tests, we empty the user's cart so we ensure we are starting with a clean slate. Before that step, we need to choose a user to log in as, and that's where this collision issue really exists.

## The Problem
You see, if one test begins executing on a user, adds some items to cart and is about to checkout and then _another_ test logs in as that user and immediately empties the cart... 💥 We've destroyed the sanctity of the first test! It can't complete it's mission because it's been compromised.

Of course, we could certainly create a single user for every individual test and that would avoid any and all collisions. However, I would posit that that is not a very scalable approach. What if we need to add 20 more tests like these next week? What if we need a hundred more by the end of the year? Let's face it, there are a lot of things to test on an e-commerce site, especially when it comes to what is definitely the most important function - placing orders. Scalability is something that often goes overlooked in software. It's because of this that I believe there are many engineers out there who almost immediately think of how a solution will scale. It's something that can bite you in the long run, and is worth at least a passing thought now and then.

I, being the only one on my "team" and the sole responder for creating these tests, did not want to get stuck making a bunch of users every time we got a new batch of criteria. So to the drawing board I went.

## The Tech
![The greatest language.](/assets/images/python.jpg)

Our test suite is built on Python using selenium and pytest at it's core. On top of these are a few pytest extensions, including pytest-xdist which is an incredibly awesome tool for running tests in parallel. There are, however, some quirks to this tool. Namely, when you execute tests in parallel using xdist, you specify the number of workers you want. These workers spin up on a delegated selection of tests. The beauty is obvious: use 2 workers, cut your run time in half, 3 and it's cut down to a third, etc. In most cases this is perfectly sufficient on it's own, but there is one quirk about these workers that became a pain for me - they can't talk to each other.

Each worker is essentially an entirely sovereign entity with it's own session, meaning any session level settings are duplicated per worker, instead of being shared. This came into the picture for me when I wanted to be able to mark a user as being "in use" in order to tell each test that that user was not available until it was released by the worker that was currently using it. So my plan was, come up with a way to share this tidbit of information between the workers - functionality that is not natively available to pytest!

## The Plan
Here's the plan in plain terms: When a test begins, it will query for an _available_ user's account id, log in as that user, mark that user as _unavailable_, execute the test, and finally _release_ that user back into the wild for another worker to pick up when finished, if needed. This would prevent any collisions because only users that are not already being used by a test will be available. This is a relatively clean, dynamic solution that allows us to simply maintain a handful of test users. Using this method, logging in as a user and emptying their cart (which would ideally already be empty) will not be a problem. 

## The Tricky Thing About Files
![sqlite](/assets/images/sqlite.PNG)

My first solution was to create a simple flat text file within the project that would store a simple data structure for holding user ids. I went with a python set since they are mutable and can only store unique items. My program would, at the beginning of a test, pull the file, read it and see what ids were marked as unavailable, then query for an id not in that set and mark that new id by storing in the set until the test was completed. Python has amazing tools for dealing with files and this actually worked! ...kind of. 

One probably obvious issue with this is the problem of file locking. Without locking the file, 2 workers are able to read it at the exact same time and both query the user database using the exact same exclusion list, thus allowing them to both pull back the same user, defeating the purpose of this solution. The obvious fix here was file locking, for which python does boast a library or two.

HOWEVER, I was at a point where I had put a lot of time into this and it was starting to feel like overkill. I didn't want to have to go learn how to wield another library just then and also have to potentially debug more issues related to it's use. Also, this flat file approach felt a little dirty to me, and that's when my research led me to something embarrassingly novel for me: SQLITE.

## The Solution
That was a big lead in to what is ultimately the payoff of this story. Thanks for sticking with me to this point! 

Sqlite is a great solution here because it easily integrates into the solution I have in mind and more importantly, it handles locking on it's own. So here are the steps I took to get this up and running:

We'll start by creating a module for executing our sqlite functions. Here we have some basic setup functions. `run_setup()` is called in the `conftest` in a fixture that is session-scoped. This does mean that each worker will call that method but you'll see why that's ok in a second.

```python
import sqlite3

from mp_selenium_app.configfiles.data_consts import SQLITE_DB


def get_connection():
    """ Get sqlite connection object """
    conn = sqlite3.connect(SQLITE_DB)
    return conn


def get_cursor(conn=None):
    """ Get sqlite cursor object """
    if conn is None:
        conn = get_connection()

    curs = conn.cursor()
    return curs


def get_connection_and_cursor():
    """ Get connection and cursor for sqlite db 
        Might seem redundant but it's fewer keystrokes later on ;)
    """
    conn = get_connection()
    curs = get_cursor(conn)
    return conn, curs


def run_setup():
    """ Initial Setup of local Sqlite3 DB """
    conn, cursor = get_connection_and_cursor()

    create_table_accounts_in_use(cursor)

    conn.commit()
```

Next we have our table creation. This will actually execute per worker, but because we have the `IF NOT EXISTS` clause, it will only happen once. After that we have some helper functions for debugging/maintenance and ease of use. Also note that the sqlite database is created in a local folder, which is shared between the workers. It wouldn't do us much good if each worker had it's own database. Fortunately that's not an issue.

```python
def create_table_accounts_in_use(cursor):
    """ Execute CREATE TABLE command for available_accounts table """
    create_accounts_in_use = "CREATE TABLE IF NOT EXISTS accounts_in_use (account_id INTEGER PRIMARY KEY);"
    cursor.execute(create_accounts_in_use)


def drop_table_accounts_in_use(cursor):
    """ Drop table command for ease of use while debugging """
    cursor.execute("""DROP TABLE IF EXISTS accounts_in_use;""")


def account_is_in_use(account_id):
    """ Returns boolean of whether or not specified account is in use """
    return select_accounts_in_use_by_id(account_id)


def get_accounts_in_use_comma_delimited_string():
    """ Returns a string of comma delimited account ids for use within a sql IN clause """
    aiu = select_all_accounts_in_use()
    aiu.append(0) # this is so the list is never empty
    return str(aiu)[1:-1]
```

Finally, we have our select functions which will query for the current accounts in use. We also have the ability to insert and delete accounts in order to keep track.

```python
def select_all_accounts_in_use():
    """ Simple select all from account table """
    curs = get_cursor()
    curs.execute(f"SELECT account_id FROM accounts_in_use;")
    accounts = []
    for row in curs.fetchall():
        accounts.append(row[0])
    return accounts


def select_accounts_in_use_by_id(account_id):
    """ Select account in use by specified id """
    curs = get_cursor()
    curs.execute(f"SELECT account_id FROM accounts_in_use WHERE account_id = {account_id}")
    return curs.fetchone()


def insert_accounts_in_use(account_id):
    """ Insert new Account Id """
    if not account_is_in_use(account_id):
        conn, curs = get_connection_and_cursor()
        curs.execute(f"""INSERT INTO accounts_in_use (account_id)
                        VALUES({account_id});""")
        conn.commit()
        conn.close()


def delete_accounts_in_use(account_id):
    """ Delete Account Id """
    conn, curs = get_connection_and_cursor()
    curs.execute(f"""DELETE FROM accounts_in_use WHERE account_id = {account_id};""")
    conn.commit()
    conn.close()
```

Next I'll show the utility module I made called `user_helper` which is essentially the endpoint for the test program to access.

```python
def setUserToInUse(accountId):
    sqlite_db.insert_accounts_in_use(accountId)


def setUserToAvailable(accountId):
    sqlite_db.delete_accounts_in_use(accountId)
```

As you can see, that one is painfully simple. All that's left is to plug all of this in! Here is a neutered sample of how I'm using this in the scope of my tests:

```python
    def test_PlacingOrder(self):
        # HERE WE GET THE USER
        companyData = self.companyData.getTestCompanyAndAccountId(TEST_COMPANY)
        
        # HERE WE SET USER TO UNAVAILABLE
        user.setUserToInUse(companyData["AccountId"])

        # some logging ...
        
        items_small = self.itemData.getTestItemsByAccountId(companyData["AccountId"],
                                                                SMALL_CART)
        orders = self.placeOrder(companyData["CompanyId"], companyData["AccountId"], items_small)
        
        # HERE WE RELEASE USER
        user.setUserToAvailable(companyData["AccountId"])

        # some asserts ...
```

But of course this is no use if we disregard the account's availability in the actual query that gets it in the first place:

```python
    def getTestCompanyAndAccountId(self, companyName, locationNote=DEFAULT_ACCOUNT_LOCATION_NOTE):
        """
        Returns a single test account id and associated company id
        """
        # THE MAGIC (Used below in the IN clause)
        accounts_in_use = sqlite_db.get_accounts_in_use_comma_delimited_string()

        self.cursor.execute(f'''
                    SELECT TOP 1 c.CompanyId, a.AccountId
                    FROM  Company c
                    JOIN Account a ON c.CompanyId = a.CompanyId
                    WHERE c.Name = '{companyName}'
                    AND a.AccountId NOT IN ({accounts_in_use})
                ''')
        record = self.cursor.fetchone()
        return {"CompanyId": record[0], "AccountId": record[1]}

```

And because sqlite handles locking on its own, there should be no need to fear multiple tests ending up logged in as the same user.

## Conclusion
This may seem like overkill, but I often prefer dynamic approaches to problems over anything more tedious in the long run. So far this solution has worked out well for us and may be a basis for other solutions in the future.

## Future Brian here (Sep 2022)
I just refactored all of this out of the test project. It wasn't the best solution. Sometimes, you need to just admit that and move on. I do think this was a fun experience but in the end, a better solution was to actually just create a user for each test (using a provided API). New users per test means NO collisions. Still, I hope this was useful!
