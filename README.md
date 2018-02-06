# Distributed Locks in MongoDB

[![Build Status](https://travis-ci.org/mongo-go/lock.svg?branch=master)](https://travis-ci.org/mongo-go/lock)
[![Go Report Card](https://goreportcard.com/badge/github.com/mongo-go/lock?)](https://goreportcard.com/report/github.com/mongo-go/lock)
[![Coverage Status](https://coveralls.io/repos/github/mongo-go/lock/badge.svg?branch=master&)](https://coveralls.io/github/mongo-go/lock?branch=master)
[![GoDoc](https://godoc.org/github.com/mongo-go/lock?status.svg)](https://godoc.org/github.com/mongo-go/lock)

This package provides a Go client for creating distributed locks in MongoDB.

## Setup
Install the package with "go get".
```
go get "github.com/mongo-go/lock"
```

In order to use it, you must have an instance of MongoDB running with a collection that can be used to store locks.
All of the examples here will assume the collection name is "locks", but you can change it to whatever you want.

#### Required Indexes
There is one index that is required in order for this package to work:
```
db.locks.createIndex( { resource: 1 }, { unique: true } )
```

#### Recommended Indexes
The following indexes are recommend to help the performance of certain queries:
```
db.locks.createIndex( { "exclusive.LockId": 1 } )
db.locks.createIndex( { "exclusive.ExpiresAt": 1 } )
db.locks.createIndex( { "shared.locks.LockId": 1 } )
db.locks.createIndex( { "shared.locks.ExpiresAt": 1 } )
```

#### Recommended Write Concern
To minimize the risk of losing locks when one or more nodes in your replica set fail, setting the write acknowledgement for the session to "majority" is recommended.

## Usage
Here is an example of how to use this package:
```go
package main

import (
	"log"

	"github.com/mongo-go/lock"
	"gopkg.in/mgo.v2"
)

func main() {
	// Create a Mongo session and set the write mode to "majority".
	mongoUrl := "youMustProvideThis"
	database := "dbName"
	collection := "collectionName"

	session, err := mgo.Dial(mongoUrl)
	if err != nil {
		log.Fatal(err)
	}
	session.SetSafe(&mgo.Safe{WMode: "majority"})

	// Create a MongoDB lock client.
	c := lock.NewClient(session, database, collection)

	lockId := "abcd1234"

	// Create an exclusive lock on resource1.
	err = c.XLock("resource1", lockId, lock.LockDetails{})
	if err != nil {
		log.Fatal(err)
	}

	// Create a shared lock on resource2.
	err = c.SLock("resource2", lockId, lock.LockDetails{}, -1)
	if err != nil {
		log.Fatal(err)
	}

	// Unlock all locks that have our lockId.
	_, err = c.Unlock(lockId)
	if err != nil {
		log.Fatal(err)
	}
}


```

## How It Works
This package can be used to create both shared and exclusive locks.
To create a lock, all you need is the name of a resource (the object that gets locked) and a lockId (lock identifier).
Multiple locks can be created with the same lockId, which makes it easy to unlock or renew a group of related locks at the same time.
Another reason for using lockIds is to ensure the only the client that creates a lock knows the lockId needed to unlock it (i.e., knowing a resource name alone is not enough to unlock it).

Here is a list of rules that the locking behavior follows
* A resource can only have one exclusive lock on it at a time.
* A resource can have multiple shared locks on it at a time [1][2].
* A resource cannot have both an exclusive lock and a shared lock on it at the same time.
* A resource can have no locks on it at all.

[1] It is possible to limit the number of shared locks that can be on a resource at a time (see the docs for [Client.SLock](https://godoc.org/github.com/mongo-go/lock#Client.SLock) for more details).
[2] A resource can't have more than one shared lock on it with the same lockId at a time.

#### Additional Features
* **TTLs**: You can optionally set a time to live (TTL) when creating a lock. If you do not set one, the lock will not have a TTL. TTLs can be renewed via the [Client.Renew](https://godoc.org/github.com/mongo-go/lock#Client.Renew) method as long as all of the locks associated with a given lockId have a TTL of at least 1 second (or no TTL at all). There is no automatic process to clean up locks that have outlived their TTL, but this package does provide a [Purger](https://godoc.org/github.com/mongo-go/lock#Purger) that can be run in a loop to accomplish this.


## Schema
Resources are the only documents stored in MongoDB. Locks on a resource are stored within the resource documents, like so:
```
{
        "resource" : "resource1",
        "exclusive" : {
                "lockId" : null,
                "owner" : null,
                "host" : null,
                "createdAt" : null,
                "renewedAt" : null,
                "expiresAt" : null,
                "acquired" : false
        },
        "shared" : {
                "count" : 1,
                "locks" : [
                        {
                                "lockId" : "abcd",
                                "owner" : "john",
                                "host" : "host.name",
                                "createdAt" : ISODate("2018-01-25T01:58:47.243Z"),
                                "renewedAt" : null,
                                "expiresAt" : null,
                                "acquired" : true,
                        }
                ]
        }
}
```
Note: shared locks are stored as an array instead of a map (keyed on lockId) so that shared lock fields can be indexed.
This helps with the performance of unlocking, renewing, and getting the status of locks.

## Tests
By default, tests expect a MongoDB instance to be running at "localhost:3000", and they write to a db "test" and randomly generated collection name.
These defaults, however, can be overwritten with environment variables.
```
export TEST_MONGO_URL="your_url"
export TEST_MONGO_DB="your_db"
```
The randomly generated collection is dropped after each test.

Run the tests from the root directory of this repo like so:
```
go test `go list ./... | grep -v "/vendor/"` --race
```
