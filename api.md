## Query API

```ts
// start query (select table)
.query("table")
// or
.query("table.index")
// or
.query("table.analyticsTable")
// or
.query("table.analyticsTable.index")

// pick one
.read("one", {
    key: ["keys", "of", "index"]
})
.read("range", {
    lt: "higher", 
    gt: "lower", 
    gte: ..., 
    lte: ...
})
.read("geo", { // custom query api
    lat: ...
    lon: ...
    range: 20
})
.read("search", {
    phrase: "something here",
    relevance: 20
})
.read("total")
.readIntersect( ["range", {...}], ["geo", {...}]) // AND
.readUnion( ["range", {...}], ["geo", {...}]) // OR

// arguments
.reverse()
.limit(50)
.where(["value", "=", "something"]) // slow af
.where((row) => row.value != "face") // slow af
.graph([
    {
        key: "posts",
        query: db.query("posts").read("one", {idx: "author", key: "{{row.id}}"}).exec("select")
    }
])

// do what? (pick one)
.exec("select", ["select", "arguments"])
.exec("upsert", {key: value})
.exec("delete");
.exec("drop");

// quick example
db.query("users").read("one", {idx: "id", key: "scott"}).exec("select");
// SELECT * FROM users WHERE id = "scott";

db.query("users").exec("upsert", {key: value});
``` 

## Transactions

```ts
db.batch([
    db.query("table").exec("upsert", {key: value}),
    db.query("table2").read("one", {idx: "id", key: "scott"}).exec("delete")
])
```

## Create Database

```ts
db.createDatabase({
    name: "databaseName",
    authActions: [
        {
            name: "authName",
            args: {
                "id:uuid": {}
            },
            call: (args, userRow, token) => {

                return token;
            }
        }
    ]
})

```

## Create Table 

```ts
db.createTable({
    database: "databaseName",
    name: "users",
    model: {
        "id:uuid": {pk: true}, // multiple primary keys are a-okay
        "name:string": {pk: true},
        "email:string": {},
        "pass:string": {lowercase: true, default: ""},
        "tags:string[]": {},
        "age:int": {max: 130, min: 13, default: 0, notNull: true},
    },
    filters: {
        select: (userData, row) => {
            row.age += 20;
            return row;

            return false; // stop select 
        },
        upsert: (userData, row) => {
            if (row.value !== "error") return false; // do not insert this row
            return row;

            return false; // stop upsert
        },
        delete: (userData, row) => {

            return false; // do not delete any rows
        }
    },
    indexes: [ // optional
        {name: "email", columns: ["email"], case_sensative: false, unique: true}
    ],
    denormalize: [ // optional
        {
            name: "posts",
            query: db.query("posts").read("one", {idx: "author", key: "{{row.id}}"}).exec("select"),
            onUpsert: (parentRow, childRow) => {
                return childRow; // modify child row

                return false; // delete child row
            },
            onDelete: (parentRow, childRow) => {

                return childRow; // modify child row

                return false; // delete child row
            },
            onDelete: false // just delete child rows on delete
        }
    ],
    analytics: [ // optional
        {
            name: "updates",
            model: {
                "date:int": {pk: true},
                "count:int": {}
            },
            start: {date: 0, count: 0}, // starting value
            rowKey: (userToken, row) => {
                var start = new Date();
                start.setHours(0,0,0,0);
                return start.getTime();
            },
            onUpdate: (userToken, prev, row) => {
                prev.count++;
                return prev;
            }
        }
    ],
    auth: { // optional
        liveStream: (userToken, row) => { // stream table updates
            return false; // no one can!
        },
        liveSync: (userToken, row) => { // push table updates dynamically
            // only super admins and users self can sync changes
            if (userToken.id === row.id || userToken.superAdmin === true) return true;
            return false;
        }
    },
    actions: [ // optional
        {
            name: "updateEmail",
            args: {
                "id:uuid": {},
                "email:string": {}
            },
            auth: (userToken, args) => { // optional

                return false;
            },
            call: (args) => {
                return db.query("users").read("one", {idx: "id", key: args.id}).exec("upsert", {email: args.email});
            },
            before: (userToken, args) => {}, // optional
            onUpdate: (userToken, args, row) => {}, // optional
            after: (userToken, args) => {} // optional
        }
    ],
    views: // same as views, except just reads data
})

```