# Using the mongo-client pod (MongoDB for beginners)

How to open a MongoDB shell inside the cluster and try some queries on the app's data. The pod is defined in [debug/mongo-client-pod.yaml](../debug/mongo-client-pod.yaml). The examples use PowerShell and the `k` alias for `kubectl`; replace `k` with `kubectl` if you don't have the alias.

## 1. Open the shell

Check the pod is running:

```powershell
k get pod mongo-client -n demo-namespace
```

If it's missing, create it:

```powershell
k apply -f k8s/debug/mongo-client-pod.yaml -n demo-namespace
```

Open a shell inside it:

```powershell
k exec -it mongo-client -n demo-namespace -- sh
```

You are now inside the pod. Connect to the database (the username and password are already environment variables there):

```sh
mongosh -u $MONGO_INITDB_ROOT_USERNAME -p $MONGO_INITDB_ROOT_PASSWORD --authenticationDatabase admin mongodb/belajar_db
```

The prompt changes to `belajar_db>`. Type `exit` once to leave `mongosh`, and again to leave the pod.

> The one-line command in the pod file's comment has double quotes inside single quotes. Windows PowerShell 5.1 strips those, so use the two-step form above.

## 2. A few concepts first

| MongoDB | Closest SQL idea | In this project |
|---|---|---|
| database | database | `belajar_db` |
| collection | table | `tutorials` |
| document | row | one tutorial, stored as JSON |
| field | column | `title`, `description`, `published` |

Each document gets an `_id` automatically. Spring Boot also adds a `_class` field. Documents in the same collection don't have to have the same fields.

## 3. Look around

```js
show dbs                          // all databases
show collections                  // collections in belajar_db
db.tutorials.countDocuments()     // how many tutorials
```

## 4. Read (find)

```js
db.tutorials.find()                                   // everything
db.tutorials.find().pretty()                          // same, easier to read
db.tutorials.findOne()                                // just the first one

db.tutorials.find({ published: true })                // filter by a field
db.tutorials.find({ published: false })

db.tutorials.find({ title: "Angular basics" })        // exact match
db.tutorials.find({ title: /angular/i })              // contains "angular", ignoring case

db.tutorials.find({ published: true }, { title: 1, _id: 0 })   // only show the title field
db.tutorials.find().sort({ title: 1 })                // A to Z (use -1 for Z to A)
db.tutorials.find().limit(3)                          // first 3 only
```

The second argument of `find` chooses which fields to show: `1` includes a field and `0` hides it.

## 5. Create (insert)

```js
db.tutorials.insertOne({
  title: "MongoDB basics",
  description: "Added from mongosh",
  published: false
})

db.tutorials.insertMany([
  { title: "Kubernetes 101", description: "Pods and Services", published: true },
  { title: "Helm 101", description: "Charts", published: false }
])
```

These documents appear in the web app straight away. Reload `http://mylocal.localhost` to see them.

## 6. Update

```js
db.tutorials.updateOne(
  { title: "MongoDB basics" },          // which document
  { $set: { published: true } }         // what to change
)

db.tutorials.updateMany(
  { published: false },
  { $set: { description: "Draft" } }
)
```

`$set` changes only the named fields. Without it, `updateOne` with a plain document is rejected.

## 7. Delete

```js
db.tutorials.deleteOne({ title: "Helm 101" })
db.tutorials.deleteMany({ published: false })
db.tutorials.deleteMany({})             // removes EVERYTHING, so be careful
```

## 8. Counting and grouping

```js
db.tutorials.countDocuments({ published: true })

db.tutorials.aggregate([
  { $group: { _id: "$published", total: { $sum: 1 } } }
])
// shows how many are published and how many are not
```

## 9. Check that the data survives a pod restart

This shows why the StatefulSet and its disk matter:

```powershell
k delete pod mongodb-0 -n demo-namespace      # Kubernetes recreates it
k get pods -n demo-namespace -w               # wait until mongodb-0 is Ready
```

Then reconnect and run `db.tutorials.countDocuments()`. The count is unchanged because the data lives on the PersistentVolumeClaim, not in the pod.

## Handy extras

- **Run one query without an interactive session.** The JavaScript has no quotes, which avoids the PowerShell quote issue:
  ```powershell
  k exec mongo-client -n demo-namespace -- sh -c 'mongosh -u $MONGO_INITDB_ROOT_USERNAME -p $MONGO_INITDB_ROOT_PASSWORD --authenticationDatabase admin mongodb/belajar_db --quiet --eval db.tutorials.countDocuments()'
  ```
- **The pod isn't managed by a Deployment.** If you delete it, nothing recreates it. Remove it when you're done with `k delete pod mongo-client -n demo-namespace`.
- **A graphical view:** [MongoDB Compass](https://www.mongodb.com/products/tools/compass) is good for beginners. Run `k port-forward svc/mongodb 27017:27017 -n demo-namespace` in one terminal and connect Compass to `mongodb://belajar:<password>@localhost:27017/?authSource=admin`. The password is in [mongodb/secret.yaml](../mongodb/secret.yaml).
