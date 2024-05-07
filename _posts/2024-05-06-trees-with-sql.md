---
layout: post
title:  "Trees with SQL"
---

## Trees with SQL

How can we use a (sql) database to encode and traverse a binary tree?

ORM: object relational mapping
Mapping of Python objects to table format to be stored inside an SQL database (and back).

### The tree format.
Edges and nodes.

### ORM

Setting up the mapping.

#### The Python code

```python

import msprime
import tskit
import sqlite3 as sql
import dataclasses

# generate a random binary tree
recombination_rate = 0.0
ts = msprime.sim_ancestry(5, recombination_rate=recombination_rate, sequence_length=10, random_seed=42)
ts.draw_svg()
```
```python
#sqlite3 has a connect() method which accepts a .db file as a destination
CONN = sql.connect('ts.db')

#once we establish our connection and assign it to CONN, we create a CURSOR
CURSOR = CONN.cursor() 

#this CURSOR object contains all the methods we will need for our ORM tasks
```

```python
@dataclasses.dataclass
class Edge:
    
    id: int
    left: float
    right: float
    parent: int
    child: int
    
    @classmethod 
    def create_table(cls):
        #use the triple quotes """ syntax to construct a SQL query
        sql = """
            CREATE TABLE IF NOT EXISTS edges (
            id INTEGER PRIMARY KEY,
            left FLOAT,
            right FLOAT,
            parent INTEGER,
            child INTEGER,
            FOREIGN KEY (parent)
                REFERENCES nodes (id)
            FOREIGN KEY (child)
                REFERENCES nodes (id));
        """
        #call CURSOR.execute and pass in our SQL query
        CURSOR.execute(sql)
        #finally, fire off the database request
        CONN.commit()

    @classmethod
    def drop_table(cls):
        #construct another SQL query to drop tables
        sql = """   
            DROP TABLE IF EXISTS edges;
        """
        CURSOR.execute(sql)
        CONN.commit()
        
    #our save() method will be called on an instance to create a SQL query
    def save(self):

        #construct a SQL query that references the database column names
        sql = """
            INSERT INTO edges (id, left, right, parent, child)
            VALUES (?, ?, ?, ?, ?);
        """

        #CURSOR.execute() will execute our SQL query with the desired values
        CURSOR.execute(sql, (self.id, self.left, self.right, self.parent, self.child))
        
        #this will fire off the SQL query using CONN.commit()
        CONN.commit()

    
    #class method create() will instantiate an object and save() it to our database
    @classmethod
    def create(cls, id, left, right, parent, child):

        #we then call the cls constructor [Edges] and pass in those values
        new_instance = cls(id, left, right, parent, child)

        #our instance method save() is what actually will query the database
        new_instance.save()

        #we return the created object so it can be used elsewhere in our code
        return new_instance

    
@dataclasses.dataclass
class Node:
    
    id: int
    flags: int
    time: float
    
    @classmethod 
    def create_table(cls):
        #use the triple quotes """ syntax to construct a SQL query
        sql = """
            CREATE TABLE IF NOT EXISTS nodes (
            id INTEGER PRIMARY KEY,
            flags INTEGER,
            time FLOAT);
        """
        #call CURSOR.execute and pass in our SQL query
        CURSOR.execute(sql)
        #finally, fire off the database request
        CONN.commit()

    @classmethod
    def drop_table(cls):
        #construct another SQL query to drop tables
        sql = """   
            DROP TABLE IF EXISTS nodes;
        """
        CURSOR.execute(sql)
        CONN.commit()
        
    #our save() method will be called on an instance to create a SQL query
    def save(self):

        #construct a SQL query that references the database column names
        sql = """
            INSERT INTO nodes (id, flags, time)
            VALUES (?, ?, ?);
        """

        #CURSOR.execute() will execute our SQL query with the desired values
        CURSOR.execute(sql, (self.id, self.flags, self.time))
        
        #this will fire off the SQL query using CONN.commit()
        CONN.commit()
        
    
    #class method create() will instantiate an object and save() it to our database
    @classmethod
    def create(cls, id, flags, time):

        #we then call the cls constructor [Pet] and pass in those values
        new_instance = cls(id, flags, time)

        #our instance method save() is what actually will query the database
        new_instance.save()

        #we return the created object so it can be used elsewhere in our code
        return new_instance
```

```python
Node.create_table()
Edge.create_table()

for node in ts.nodes():
    # create node(id, flags, time)
    Node.create(node.id, node.flags, node.time)

for edge in ts.edges():
    # create edge(id, left, right, parent, child)
    Edge.create(edge.id, edge.left, edge.right, edge.parent, edge.child)
```