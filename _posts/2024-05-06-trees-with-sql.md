---
layout: post
title:  "Trees with SQL"
---

## Trees with SQL

Aim of this post: 
* Introduction to ORM (object relational mapping): using ORM, Python objects can be mapped to table
format to be stored inside an SQL database (and back).
* Demonstrate how can we use a (sql) database to encode and traverse a binary tree.

### The tree format.
To encode a binary tree we store the following information: `left_child_array` and `right_sib_array`.
These arrays contain the indices of the respective left child and right sib of each node.
Note that if `left_child_array[i] = j` and `right_sib_array[j] = k`, then `j` and `k` are the children
of `i`. The following code snippets generates a random binary tree:

```python
import numpy as np

# generate a random binary tree
def generate_random_binary_tree(num_leaves, rng):
    left_child_array = np.full(num_leaves * 2 - 1, -1, dtype=int)  # Initialize left child array
    right_sib_array = np.full(num_leaves * 2 - 1, -1, dtype=int)   # Initialize right sibling array

    extant = list(range(num_leaves))
    internal_nodes = list(range(num_leaves, left_child_array.size)) 
    
    while len(extant) > 1:
        # Randomly shuffle the nodes
        rng.shuffle(extant)
        rng.shuffle(internal_nodes)
        
        # sample left child
        left_child = extant.pop()
        # sample parent
        parent = internal_nodes.pop()
        left_child_array[parent] = left_child
        # sample right sib
        right_sib = extant.pop()
        right_sib_array[left_child] = right_sib
        extant.append(parent)
    assert len(internal_nodes) == 0

    return left_child_array, right_sib_array, extant[0]

num_leaves = 10
rng = numpy.random.default_rng(42)
left_child_array, right_sib_array, root = generate_random_binary_tree(num_leaves, rng)
```

<p align="center">
  <img src="../_figures/tree_with_sql/tree_traversal.png" />
</p>


### ORM

Object relational mapping is a technique that allows you to interact with a relational database using
an object oriented approach with languages like Python.
Here I'll stick to the basics using *sqlite3* and extend some ideas introduced in
[this blogpost](https://medium.com/@joeylee08/object-relational-mapping-from-python-to-sql-and-back-cd629eca0060) by Joseph Lee.

The first bit of code is needed to establish a connection (and create) a sql database. Let's make abstraction of how
this works for now. In the subsequent code snippets you'll find that we use the `CURSOR`
to execute SQL-command, and `CONN` to ensure that the consequence of running a particular command is stored within
the database.

```python
import sqlite3 as sql
import dataclasses

#sqlite3 has a connect() method which accepts a .db file as a destination
CONN = sql.connect('tree.db')
#once we establish our connection and assign it to CONN, we create a CURSOR
CURSOR = CONN.cursor()
#this CURSOR object contains all the methods we will need for our ORM tasks
```
The database will encode both information on the edges and the nodes of the tree.
The rows in the edge table will
have the following structure: `(id, parent, child)`. Here, both parent and child
are nodes in the tree which thus will occur in the nodes table. Each row in the nodes table
will have the following entries: `(id, flags, rank, left_child, right_sib)`. 
The `flag` is a simple boolean indicating whether a node is a leaf (`True`) or not (`False`).
The `rank` allows us to easily identify the root. This is the node with the highest rank.

The following code creates a class for both nodes and edges and allows us to not only create
a table in the database for such class, but more importantly translate each such object into
a corresponding entry in the correct table. Note that the `create_table` method for the `Edge`
class has the following sql-specifier: `FOREIGN KEY (parent)`. This generates the "relational"
part of the relational database that we will be generating. More specifically, this indicates
that both the `parent` and `child` column will always correspond to the `PRIMARY KEY`, or unique
identifier of a node in the nodes table.

```python
@dataclasses.dataclass
class Edge:
    
    id: int
    parent: int
    child: int
    
    @classmethod 
    def create_table(cls):
        #use the triple quotes """ syntax to construct a SQL query
        sql = """
            CREATE TABLE IF NOT EXISTS edges (
            id INTEGER PRIMARY KEY,
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
            INSERT INTO edges (id, parent, child)
            VALUES (?, ?, ?);
        """

        #CURSOR.execute() will execute our SQL query with the desired values
        CURSOR.execute(sql, (self.id, self.parent, self.child))
        
        #this will fire off the SQL query using CONN.commit()
        CONN.commit()

    
    #class method create() will instantiate an object and save() it to our database
    @classmethod
    def create(cls, id, parent, child):

        #we then call the cls constructor [Edges] and pass in those values
        new_instance = cls(id, parent, child)

        #our instance method save() is what actually will query the database
        new_instance.save()

        #we return the created object so it can be used elsewhere in our code
        return new_instance

    
@dataclasses.dataclass
class Node:
    
    id: int
    flags: int
    rank: int
    left_child: int
    right_sib: int
    
    @classmethod 
    def create_table(cls):
        #use the triple quotes """ syntax to construct a SQL query
        sql = """
            CREATE TABLE IF NOT EXISTS nodes (
            id INTEGER PRIMARY KEY,
            flags INTEGER,
            rank INTEGER,
            left_child INTEGER,
            right_sib INTEGER
            );
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
            INSERT INTO nodes (id, flags, rank, left_child, right_sib)
            VALUES (?, ?, ?, ?, ?);
        """

        #CURSOR.execute() will execute our SQL query with the desired values
        CURSOR.execute(sql, (self.id, self.flags, self.rank, self.left_child, self.right_sib))
        
        #this will fire off the SQL query using CONN.commit()
        CONN.commit()
        
    
    #class method create() will instantiate an object and save() it to our database
    @classmethod
    def create(cls, id, flags, rank, left_child, right_sib):

        #we then call the cls constructor [Pet] and pass in those values
        new_instance = cls(id, flags, rank, left_child, right_sib)

        #our instance method save() is what actually will query the database
        new_instance.save()

        #we return the created object so it can be used elsewhere in our code
        return new_instance
```
Now that we've done all this preperation work, we can actually create our database
by performing a tree traversal and generating the necessary nodes and edges as we
encounter them.

```python

Node.create_table()
Edge.create_table()
edge_count = 0
rank_array = np.zeros_like(left_child_array)
rank_array[-1] = num_leaves - 1

stack = [root]
while stack:
    node = stack.pop()
    left_child = left_child_array[node]
    if left_child != -1:
        Edge.create(int(edge_count), int(node), int(left_child))
        edge_count += 1
        right_sib = right_sib_array[left_child]
        rank_array[left_child] = rank_array[node] - 1
        stack.append(left_child)
        flag = 0
        while right_sib != -1:
            rank_array[right_sib] = rank_array[left_child]
            Edge.create(edge_count, node, right_sib)
            edge_count += 1
            stack.append(right_sib)
            right_sib = right_sib_array[right_sib]
    else:
        flag = 0
    Node.create(int(node), flag, int(rank_array[node]), int(left_child), int(right_sib_array[node]))
```

### Identifying the samples subtending a node using a recursive SQL query

If you look closely you can see that the following SQL query actually performs
a very similar tree traversal algorithm as the one we just did in python. Except
here we make use of a **recursive common table expression (CTE)**.
The first section identifies the internal node to start from and takes this as the new
root of the tree traversal. The second bit, after `UNION ALL` does the actual recursion.
We then finally filter out the leaf nodes we encountered, as these are the ones we are interested in.

```python
internal_node_id = 16
sql = f"""WITH RECURSIVE BinaryTreeTraversal AS (
    SELECT id, rank, left_child, right_sib
    FROM nodes
    WHERE id = {internal_node_id} -- Start with the specified internal node
    
    UNION ALL
    
    SELECT t.id, t.rank, t.left_child, t.right_sib
    FROM nodes AS t
    JOIN BinaryTreeTraversal AS p 
    ON ((t.id = p.left_child)
       OR (t.id = p.right_sib AND p.id != {internal_node_id}))
)
, LeafNodes AS (
    SELECT id, rank
    FROM BinaryTreeTraversal
    WHERE left_child = -1 -- Select nodes without left child, i.e., leaf nodes
)
SELECT id
FROM LeafNodes;
"""
CURSOR.execute(sql)
results = CURSOR.fetchall()
results
> "[(5,), (3,), (4,), (0,), (8,), (6,), (1,)]"
```


Such an algorithm can for example be used to identify those sequences that carry a particular mutation.
This demonstrates that sequence data can be very efficiently represented using trees as we only need to
store a single mutation/value above the node subtending all samples hit by that mutation.

