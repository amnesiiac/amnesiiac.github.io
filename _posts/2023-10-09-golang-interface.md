---
layout: post
title: "interface type (golang)"
author: "melon"
date: 2023-10-09 20:49
categories: "2023"
tags:
  - golang
---

the article introduces: what is golang interface type; why use interface type; how to use interface type.

<hr>

### # basic interfaces type usage introduction
the below code create two customized data struct (both satisfy the interface: fmt.Stringer),
thus the instance of the data struct can be filled in anywhere accepting fnt.Stringer type.

```text
package main

import (
    "fmt"
    "strconv"
    "log"
)

type Book struct {                                          // satisfies the fmt.Stringer interface
    Title  string
    Author string
}
func (b Book) String() string {                             // Book.String method: print info
    return fmt.Sprintf("Book: %s - %s", b.Title, b.Author)
}

type Count int                                              // satisfiy the fmt.Stringer interface
func (c Count) String() string {                            // Count's String method
    return strconv.Itoa(int(c))
}

func WriteLog(s fmt.Stringer) {                             // interface utilize func, with interface type as param
    log.Print(s.String())
}

func main() {
    book := Book{"Alice in Wonderland", "Lewis Carrol"}     // define book obj satisfy the interface
    WriteLog(book)
    count := Count(3)                                       // define count obj satisfy the interface
    WriteLog(count)
}
```

compile & execute the program:

```text
$ go build test.go && ./test
$ Book: Alice in Wonderland - Lewis Carrol
$ 3
```

<hr>

### # why interfaces type are useful, and when they are needed?
1 the interface can help reduce duplication or boilerplate code (meta programming):

```text
package main

import (
    "bytes"
    "encoding/json"
    "io"
    "log"
    "os"
)

type Customer struct {
    Name string
    Age  int
}

func (c *Customer) WriteJSON(w io.Writer) error {   // accept io.Write as param, to marshal customer obj as json
    js, err := json.Marshal(c)
    if err != nil {
        return err
    }
    _, err = w.Write(js)
    return err
}

func main() {
    c := &Customer{Name: "Alice", Age: 21}

    var buf bytes.Buffer                            // 1 write to a buffer
    err := c.WriteJSON(&buf)                        // bytes.Buffer satisfy the WriteJSON interface param type
    if err != nil {
        log.Fatal(err)
    }

    f, err := os.Create("/tmp/customer")            // 2 write to a file
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()
    err = c.WriteJSON(f)                            // os.File satisfy the WriteJSON interface param type
    if err != nil {
        log.Fatal(err)
    }
}
```

in above program, both the bytes.Buffer & os.File satisfy the WriteJSON param interface type.
that is, all obj satisfy the interface def can be used in any place accepting the interface type.

<p style="margin-bottom: 20px;"></p>

2 interfaces make it easier to use mocks instead of real objects in unit tests  
let's say you run a shop, and you store information of num of customers and sales in a postgresql db.
here comes the need to calculate the sales rate (i.e. sales per customer) for the past 24 hours:

```text
package main

import (
    "fmt"
    "log"
    "time"
    "database/sql"
    _ "github.com/lib/pq"
)

type ShopDB struct {                                                        // declare shopdb
    *sql.DB
}

func (sdb *ShopDB) CountCustomers(since time.Time) (int, error) {           // shopdb method: count customers
    var count int
    err := sdb.QueryRow("SELECT count(*) FROM customers WHERE timestamp > $1", since).Scan(&count)
    return count, err
}

func (sdb *ShopDB) CountSales(since time.Time) (int, error) {               // shopdb method: count sales
    var count int
    err := sdb.QueryRow("SELECT count(*) FROM sales WHERE timestamp > $1", since).Scan(&count)
    return count, err
}

func main() {
    db, err := sql.Open("postgres", "postgres://user:pass@localhost/db")    // connect to localhost sql db service
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    shopDB := &ShopDB{db}                                                   // create shopdb obj
    sr, err := calculateSalesRate(shopDB)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf(sr)
}

func calculateSalesRate(sdb *ShopDB) (string, error) {                      // compute sales / customers
    since := time.Now().Add(-24 * time.Hour)
    sales, err := sdb.CountSales(since)
    if err != nil {
        return "", err
    }
    customers, err := sdb.CountCustomers(since)
    if err != nil {
        return "", err
    }
    rate := float64(sales) / float64(customers)
    return fmt.Sprintf("%.2f", rate), nil
}
```

what if we want to create a unittest for the calculateSalesRate() to make sure that the math logic is working
correctly? currently there's a pain in ut implementation: for validation of the calculated data, we need to
setup a test instance of the postgresql db with start/teardown script to scaffold the db with some dummy data,
that's quite lot of work!

luckily, with the help of of golang interface, the ut implementation is at your fingertips.
consider the following diff for adding interface support for calculateSalesRate() under test:

```text
diff --git a/test.go b/test-1.go
index a2805c9..586741c 100644
--- a/main.go
+++ b/main1.go
@@ -8,6 +8,11 @@ import (
     _ "github.com/lib/pq"
 )

+type ShopModel interface {
+    CountCustomers(time.Time) (int, error)
+    CountSales(time.Time) (int, error)
+}
+
 type ShopDB struct {
     *sql.DB
 }
@@ -39,15 +44,15 @@ func main() {
     fmt.Printf(sr)
 }

-func calculateSalesRate(sdb *ShopDB) (string, error) {
+func calculateSalesRate(sm ShopModel) (string, error) {
     since := time.Now().Add(-24 * time.Hour)

-    sales, err := sdb.CountSales(since)
+    sales, err := sm.CountSales(since)
     if err != nil {
         return "", err
     }

-    customers, err := sdb.CountCustomers(since)
+    customers, err := sm.CountCustomers(since)
     if err != nil {
         return "", err
     }
```

thus, all obj meet the interface definition can be passed to calculateSalesRate() for ut purpose.

```text
package main

import (
    "testing"
    "time"
)

type MockShopDB struct{}                                         // declare the mock shopdb

func (m *MockShopDB) CountCustomers(_ time.Time) (int, error) {  // make mock ret val for ut
    return 1000, nil
}

func (m *MockShopDB) CountSales(_ time.Time) (int, error) {      // make mock ret val for ut
    return 333, nil
}

func TestCalculateSalesRate(t *testing.T) {                      // ut naming convention: Testxxx
    m := &MockShopDB{}                                           // init the mock shopdb
    sr, err := calculateSalesRate(m)                             // pass to func under ut
    if err != nil {
        t.Fatal(err)
    }
    exp := "0.33"                                                // check return value based on mock input
    if sr != exp {
        t.Fatalf("got %v; expected %v", sr, exp)
    }
}
```

<p style="margin-bottom: 20px;"></p>

3 as an architectural tool, to help enforce decoupling between parts of your codebase  
the empty interface type essentially describes no methods: it has no rules, thus, any and every obj satisfies
this interface.

<hr>

### # empty interfaces
below is a toy program illustrate the mechanism & usage of empty interfaces.

```text
package main

import "fmt"

func main() {
    person := make(map[string]interface{}, 0)  // define a map string -> interface{}, thus any obj can serve as value

    person["name"] = "Alice"                   // str -> string
    person["age"] = 21                         // str -> int
    person["height"] = 167.64                  // str -> float

    person["age"] = person["age"] + 1          // compile err: invalid operation, mismatched types interface {} and int
    age, ok := person["age"].(int)             // convert type before usage
    if !ok {
        log.Fatal("could not assert value to int")
        return
    }
    person["age"] = age + 1

    fmt.Printf("%+v", person)
}
```

the empty interface is useful in situations where you need to accept and work with unpredictable or user-defined types.
you'll see it used in places throughout the standard library, such as: gob.Encode, fmt.Print and template.Execute functions.

<hr>

### # the any identifier: replica of interfaces{}
go 1.18 introduced a new predeclared identifier called any, as an alias for the empty interface interface{},
the identifier is just a syntactic sugar: any is equivalent as interface{} in all ways.  
so map[string]any is exactly the same as map[string]interface{} in terms of it's behavior.
it's shorter and saves typing, and more clearly conveys to the reader that you can use any type here.
