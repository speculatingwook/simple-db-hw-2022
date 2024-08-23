# 6.5830/6.5831 Lab 1: SimpleDB

**Assigned:** Wednesday September 14, 2022

**Due:** Wednesday September 28, 2022 by 11:59 PM ET

In the lab assignments in 6.5830/6.5831 you will write a basic database management system called SimpleDB. For this lab, you will focus on implementing the core modules required to access stored data on disk; in future labs, you will add support for various query processing operators, as well as transactions, locking, and concurrent queries.

SimpleDB is written in Java. We have provided you with a set of mostly unimplemented classes and interfaces. You will need to write the code for these classes. We will grade your code by running a set of system tests written using [JUnit](http://junit.sourceforge.net/). We have also provided a number of unit tests, which we will not use for grading but that you may find useful in verifying that your code works.

The remainder of this document describes the basic architecture of SimpleDB, gives some suggestions about how to start coding, and discusses how to hand in your lab.

We **strongly recommend** that you start as early as possible on this lab. It requires you to write a fair amount of code!

## 0. Environment Setup

**Start by downloading the code for lab 1 from the course GitHub repository by following the instructions [here](https://github.com/MIT-DB-Class/simple-db-hw-2022).**

These instructions are written for Athena or any other Unix-based platform (e.g., Linux, macOS, etc.) Because the code is written in Java, it should work under Windows as well, although the directions in this document may not apply.

SimpleDB uses Java 11 by default. If you are running an older version of Java, you will need to update the following line in `build.gradle`:

```groovy
sourceCompatibility = '11'
```

**Our autograder uses Java 11, so please do not use any features newer than Java 11 (you will not need them for the lab).**

You can install a JDK as follows:
- On macOS: `brew install openjdk@11`
- On Ubuntu (or WSL): `sudo apt install openjdk-11-jdk`
- On Windows: See this [guide](https://docs.microsoft.com/en-us/java/openjdk/install).

## 1. Getting started

SimpleDB uses the Gradle build tool to compile the code and run tests. Gradle is similar to Ant, but the build file is written in Groovy and is somewhat better suited to Java code.

To help you during development, we have provided a set of unit tests in addition to the end-to-end tests that we use for grading. These are by no means comprehensive, and you should not rely on them exclusively to verify the correctness of your project (put those 6.1040 (formerly 6.170) skills to use!).

To run the unit tests use the `test` Gradle task:

```
$ cd [project-directory]
$ # run all unit tests
$ ./gradlew test
$ # run a specific unit test
$ ./gradlew test --tests "simpledb.TupleTest"
```

You should see output similar to:

```
BUILD SUCCESSFUL in 1s
3 actionable tasks: 3 executed
```

For more details about how to use Gradle, see the [Gradle Command-Line Interface](https://docs.gradle.org/current/userguide/command_line_interface.html) documentation. The quick reference table below should be sufficient for working on the labs.

Command | Description
--- | ---
./gradlew build | Compile all code in the project
./gradlew test | Compile and run all unit tests
./gradlew test --tests "testname" | Run the unit test named `testname`
./gradlew systemTest | Compile and run all system tests
./gradlew systemTest --tests "testname" | Compile and run the system test named `testname`

If you are using an IDE, you can also run these tasks from within the IDE. For example, in IntelliJ IDEA, you can right-click on the `build.gradle` file and select "Run Gradle Tasks".

### 1.1. Running end-to-end tests

We have also provided a set of end-to-end tests that will eventually be used for grading. These tests are structured as JUnit tests that live in the `test/simpledb/systemtest` directory. To run all the system tests, use the `systemTest` Gradle task:

```
$ ./gradlew systemTest
```

### 1.2. Creating dummy tables

It is likely you'll want to create your own tests and your own data tables to test your own implementation of SimpleDB. You can create any `.txt` file and convert it to a `.dat` file in SimpleDB's `HeapFile` format using the command:

```
$ java -cp build/libs/simpledb.jar simpledb.SimpleDb convert file.txt N
```

where `file.txt` is the name of the file and `N` is the number of columns in the file. Notice that `file.txt` has to be in the following format:

```
int1,int2,...,intN
int1,int2,...,intN
int1,int2,...,intN
int1,int2,...,intN
```

...where each `intN` is a non-negative integer.

To view the contents of a table, use the `print` command:

```
$ java -cp build/libs/simpledb.jar simpledb.SimpleDb print file.dat N
```

where `file.dat` is the name of a table created with the `convert` command, and `N` is the number of columns in the file.

### 1.3. Working with an IDE

We recommend using IntelliJ IDEA, VSCode, or Eclipse for this project. These IDEs have good support for Gradle projects.

For IntelliJ IDEA:
1. Choose "Open" from the start screen or File > Open from the menu.
2. Select the project directory.
3. In the "Import Project from Gradle" dialog, select "Use gradle 'wrapper' task configuration".

For VSCode:
1. Install the "Java Extension Pack" and the "Gradle for Java" extensions.
2. Open the project folder in VSCode.

For Eclipse:
1. Choose File > Import > Gradle > Existing Gradle Project.
2. Select the project directory and click "Finish".

## 2. SimpleDB Architecture and Implementation Guide

SimpleDB consists of:

* Classes that represent fields, tuples, and tuple schemas;
* Classes that apply predicates and conditions to tuples;
* One or more access methods (e.g., heap files) that store relations on disk and provide a way to iterate through tuples of those relations;
* A collection of operator classes (e.g., select, join, insert, delete, etc.) that process tuples;
* A buffer pool that caches active tuples and pages in memory and handles concurrency control and transactions (neither of which you need to worry about for this lab); and,
* A catalog that stores information about available tables and their schemas.

SimpleDB does not include many things that you may think of as being a part of a "database." In particular, SimpleDB does not have:

* (In this lab), a SQL front end or parser that allows you to type queries directly into SimpleDB. Instead, queries are built up by chaining a set of operators together into a hand-built query plan (see Section 2.7).
* Views.
* Data types except integers and fixed length strings.
* (In this lab) Query optimizer.
* (In this lab) Indices.

In the rest of this Section, we describe each of the main components of SimpleDB that you will need to implement in this lab. You should use the exercises in this discussion to guide your implementation. This document is by no means a complete specification for SimpleDB; you will need to make decisions about how to design and implement various parts of the system.

### 2.1. The Database Class

The Database class provides access to a collection of static objects that are the global state of the database. In particular, this includes methods to access the catalog (the list of all the tables in the database), the buffer pool (the collection of database file pages that are currently resident in memory), and the log file. You will not need to worry about the log file in this lab. We have implemented the Database class for you. You should take a look at this file as you will need to access these objects.

### 2.2. Fields and Tuples

Tuples in SimpleDB are quite basic. They consist of a collection of `Field` objects, one per field in the `Tuple`. `Field` is an interface that different data types (e.g., integer, string) implement. `Tuple` objects are created by the underlying access methods (e.g., heap files, or B-trees), as described in the next section. Tuples also have a type (or schema), called a _tuple descriptor_, represented by a `TupleDesc` object. This object consists of a collection of `Type` objects, one per field in the tuple, each of which describes the type of the corresponding field.

### Exercise 1

**Implement the skeleton methods in:**
* src/java/simpledb/storage/TupleDesc.java
* src/java/simpledb/storage/Tuple.java

At this point, your code should pass the unit tests `TupleTest` and `TupleDescTest`.

### 2.3. Catalog

The catalog (class `Catalog` in SimpleDB) consists of a list of the tables and schemas of the tables that are currently in the database. You will need to support the ability to add a new table, as well as getting information about a particular table. Associated with each table is a `TupleDesc` object that allows operators to determine the types and number of fields in a table.

The global catalog is a single instance of `Catalog` that is allocated for the entire SimpleDB process. The global catalog can be retrieved via the method `Database.getCatalog()`, and the same goes for the global buffer pool (using `Database.getBufferPool()`).

### Exercise 2

**Implement the skeleton methods in:**
* src/java/simpledb/common/Catalog.java

At this point, your code should pass the unit tests in `CatalogTest`.

### 2.4. BufferPool

The buffer pool (class `BufferPool` in SimpleDB) is responsible for caching pages in memory that have been recently read from disk. All operators read and write pages from various files on disk through the buffer pool. It consists of a fixed number of pages, defined by the `numPages` parameter to the `BufferPool` constructor. In later labs, you will implement an eviction policy. For this lab, you only need to implement the constructor and the `BufferPool.getPage()` method used by the SeqScan operator. The BufferPool should store up to `numPages` pages. For this lab, if more than `numPages` requests are made for different pages, then instead of implementing an eviction policy, you may throw a `DbException`. In future labs you will be required to implement an eviction policy.

The `Database` class provides a static method, `Database.getBufferPool()`, that returns a reference to the single BufferPool instance for the entire SimpleDB process.

### Exercise 3

**Implement the `getPage()` method in:**
* src/java/simpledb/storage/BufferPool.java

We have not provided unit tests for `BufferPool`. The functionality you implemented will be tested in the implementation of `HeapFile` below. You should use the `DbFile.readPage` method to access pages of a `DbFile`.

### 2.5. HeapFile access method

Access methods provide a way to read or write data from disk that is arranged in a specific way. Common access methods include heap files (unsorted files of tuples) and B-trees; for this assignment, you will only implement a heap file access method, and we have written some of the code for you.

A `HeapFile` object is arranged into a set of pages, each of which consists of a fixed number of bytes for storing tuples, (defined by the constant `BufferPool.DEFAULT_PAGE_SIZE`), including a header. In SimpleDB, there is one `HeapFile` object for each table in the database. Each page in a `HeapFile` is arranged as a set of slots, each of which can hold one tuple (tuples for a given table in SimpleDB are all of the same size). In addition to these slots, each page has a header that consists of a bitmap with one bit per tuple slot. If the bit corresponding to a particular tuple is 1, it indicates that the tuple is valid; if it is 0, the tuple is invalid (e.g., has been deleted or was never initialized.)

SimpleDB stores heap files on disk in more or less the same format they are stored in memory. Each file consists of page data arranged consecutively on disk. Each page consists of one or more bytes representing the header, followed by the _page size_ bytes of actual page content. Each tuple requires _tuple size_ * 8 bits for its content and 1 bit for the header. Thus, the number of tuples that can fit in a single page is:

```
tuples per page = floor((page size * 8) / (tuple size * 8 + 1))
```

Where _tuple size_ is the size of a tuple in the page in bytes. The idea here is that each tuple requires one additional bit of storage in the header. We compute the number of bits in a page (by multiplying page size by 8), and divide this quantity by the number of bits in a tuple (including this extra header bit) to get the number of tuples per page. The floor operation rounds down to the nearest integer number of tuples (we do not want to store partial tuples on a page!)

Once we know the number of tuples per page, the number of bytes required to store the header is simply:

```
header bytes = ceiling(tuples per page / 8)
```

The ceiling operation rounds up to the nearest integer number of bytes (we never store less than a full byte of header information.)

The low (least significant) bits of each byte represents the status of the slots that are earlier in the file. Hence, the lowest bit of the first byte represents whether or not the first slot in the page is in use. The second lowest bit of the first byte represents whether or not the second slot in the page is in use, and so on. Also, note that the high-order bits of the last byte may not correspond to a slot that is actually in the file, since the number of slots may not be a multiple of 8.

### Exercise 4

**Implement the skeleton methods in:**
* src/java/simpledb/storage/HeapPageId.java
* src/java/simpledb/storage/RecordId.java
* src/java/simpledb/storage/HeapPage.java

Although you will not use them directly in Lab 1, we ask you to implement `getNumUnusedSlots()` and `isSlotUsed()` in `HeapPage`. These require pushing around bits in the page header. You may find it helpful to look at the other methods that have been provided in `HeapPage` or in `src/simpledb/HeapFileEncoder.java` to understand the layout of pages.

You will also need to implement an `Iterator` over the tuples in the page, which may involve an auxiliary class or data structure.

At this point, your code should pass the unit tests in `HeapPageIdTest`, `RecordIDTest`, and `HeapPageReadTest`.

After you have implemented `HeapPage`, you will write methods for `HeapFile` in this lab to calculate the number of pages in a file and to read a page from the file. You will then be able to fetch tuples from a file stored on disk.

### Exercise 5

**Implement the skeleton methods in:**
* src/java/simpledb/storage/HeapFile.java

To read a page from disk, you will first need to calculate the correct offset in the file. Hint: you will need random access to the file in order to read and write pages at arbitrary offsets. You should not call `BufferPool` methods when reading a page from disk.

You will also need to implement the `HeapFile.iterator()` method, which should iterate through the tuples of each page in the HeapFile. The iterator must use the `BufferPool.getPage()` method to access pages in the `HeapFile`. This method loads the page into the buffer pool and will eventually be used (in a later lab) to implement locking-based concurrency control and recovery. Do not load the entire table into memory on the `open()` call -- this will cause an out of memory error for very large tables.

At this point, your code should pass the unit tests in `HeapFileReadTest`.

### 2.6. Operators

Operators are responsible for the actual execution of the query plan. They implement the operations of the relational algebra. In SimpleDB, operators are iterator based; each operator implements the `DbIterator` interface.

Operators are connected together into a plan by passing lower-level operators into the constructors of higher-level operators, i.e., by "chaining them together." Special access method operators at the leaves of the plan are responsible for reading data from the disk (and hence do not have any operators below them).

At the top of the plan, the program interacting with SimpleDB simply calls `getNext()` on the root operator; this operator then calls `getNext()` on its children, and so on, until these leaf operators are called. They fetch tuples from disk and pass them up the tree (as return arguments to `getNext()`); tuples propagate up the plan in this way until they are output at the root or combined or rejected by another operator in the plan.

For this lab, you will only need to implement one SimpleDB operator.

### Exercise 6

**Implement the skeleton methods in:**
* src/java/simpledb/execution/SeqScan.java

This operator sequentially scans all of the tuples from the pages of the table specified by the `tableid` in the constructor. This operator should access tuples through the `DbFile.iterator()` method.

At this point, you should be able to complete the ScanTest system test. Good work!

You will fill in other operators in subsequent labs.

### 2.7. A simple query

The purpose of this section is to illustrate how these various components are connected together to process a simple query.

Suppose you have a data file, "some_data_file.txt", with the following contents:

```
1,1,1
2,2,2 
3,4,4
```

You can convert this into a binary file that SimpleDB can query as follows:

```bash
java -cp build/libs/simpledb.jar simpledb.SimpleDb convert some_data_file.txt 3
```

Here, the argument "3" tells convert that the input has 3 columns.

The following code implements a simple selection query over this file. This code is equivalent to the SQL statement `SELECT * FROM some_data_file`.

```java
package simpledb;
import java.io.*;

public class test {

    public static void main(String[] argv) {

        // construct a 3-column table schema
        Type types[] = new Type[]{ Type.INT_TYPE, Type.INT_TYPE, Type.INT_TYPE };
        String names[] = new String[]{ "field0", "field1", "field2" };
        TupleDesc descriptor = new TupleDesc(types, names);

        // create the table, associate it with some_data_file.dat
        // and tell the catalog about the schema of this table.
        HeapFile table1 = new HeapFile(new File("some_data_file.dat"), descriptor);
        Database.getCatalog().addTable(table1, "test");

        // construct the query: we use a simple SeqScan, which spoonfeeds
        // tuples via its iterator.
        TransactionId tid = new TransactionId();
        SeqScan f = new SeqScan(tid, table1.getId());

        try {
            // and run it
            f.open();
            while (f.hasNext()) {
                Tuple tup = f.next();
                System.out.println(tup);
            }
            f.close();
            Database.getBufferPool().transactionComplete(tid);
        } catch (Exception e) {
            System.out.println ("Exception : " + e);
        }
    }
}
```

The table we create has three integer fields. To express this, we create a `TupleDesc` object and pass it an array of `Type` objects, and optionally an array of `String` field names. Once we have created this `TupleDesc`, we initialize a `HeapFile` object representing the table stored in `some_data_file.dat`. Once we have created the table, we add it to the catalog. If this were a database server that was already running, we would have this catalog information loaded. We need to load it explicitly to make this code self-contained.

Once we have finished initializing the database system, we create a query plan. Our plan consists only of the `SeqScan` operator that scans the tuples from disk. In general, these operators are instantiated with references to the appropriate table (in the case of `SeqScan`) or child operator (in the case of e.g. Filter). The test program then repeatedly calls `hasNext` and `next` on the `SeqScan` operator. As tuples are output from the `SeqScan`, they are printed out on the command line.

We **strongly recommend** you try this out as a fun end-to-end test that will help you get experience writing your own test programs for simpledb. You should create the file "test.java" in the `src/java/simpledb` directory with the code above, and you should add some "import" statement above the code, and place the `some_data_file.dat` file in the top level directory. Then run:

```bash
./gradlew build
java -cp build/libs/simpledb.jar simpledb.test
```

Note that `gradlew build` compiles `test.java` and generates a new jar file that contains it.

## 3. Logistics

You must submit your code (see below) as well as a short (2 pages, maximum) writeup describing your approach. This writeup should:

* Describe any design decisions you made. These may be minimal for Lab 1.
* Discuss and justify any changes you made to the API.
* Describe any missing or incomplete elements of your code.
* Describe how long you spent on the lab, and whether there was anything you found particularly difficult or confusing.

### 3.1. Collaboration

This lab should be manageable for a single person, but if you prefer to work with a partner, this is also OK. Larger groups are not allowed. Please indicate clearly who you worked with, if anyone, on your individual writeup.

### 3.2. Submitting your assignment

We will be using Gradescope to autograde all programming assignments. You should have all been invited to the class instance; if not, please check Piazza for an invite code. If you are still having trouble, let us know and we can help you set up. You may submit your code multiple times before the deadline; we will use the latest version as determined by Gradescope. Place the write-up in a file called `lab1-writeup.txt` with your submission.

If you are working with a partner, only one person needs to submit to Gradescope. However, make sure to add the other person to your group. Also note that each member must have their own writeup. Please add your Kerberos username to the file name and in the writeup itself (e.g., `lab1-writeup-username1.txt` and `lab1-writeup-username2.txt`).

The easiest way to submit to Gradescope is with `.zip` files containing your code. On Linux/macOS, you can do so by running the following command:

```bash
$ zip -r submission.zip src/ lab1-writeup.txt build.gradle settings.gradle

# If you are working with a partner:
$ zip -r submission.zip src/ lab1-writeup-username1.txt lab1-writeup-username2.txt build.gradle settings.gradle
```

### 3.3. Submitting a bug

Please submit (friendly!) bug reports to [6.5830-staff@mit.edu](mailto:6.5830-staff@mit.edu). When you do, please try to include:

* A description of the bug.
* A `.java` file we can drop in the `src/java/simpledb` directory, compile, and run.
* A `.txt` file with the data that reproduces the bug. We should be able to convert it to a `.dat` file using HeapFileEncoder.

If you are the first person to report a particular bug in the code, we will give you a candy bar!

### 3.4 Grading

75% of your grade will be based on whether or not your code passes the system test suite we will run over it. These tests will be a superset of the tests we have provided. Before handing in your code, you should make sure it produces no errors (passes all of the tests) from both `./gradlew test` and `./gradlew systemTest`.

**Important:** before testing, Gradescope will replace your `build.gradle`, `settings.gradle`, and the entire contents of the `test` directory with our version of these files. This means you cannot change the format of `.dat` files! You should also be careful changing our APIs. You should test that your code compiles with the unmodified tests.

You should get immediate feedback and error outputs for failed tests (if any) from Gradescope after submission. The score given will be your grade for the autograded portion of the assignment. An additional 25% of your grade will be based on the quality of your writeup and our subjective evaluation of your code. This part will also be published on Gradescope after we finish grading your assignment.

We had a lot of fun designing this assignment, and we hope you enjoy hacking on it!