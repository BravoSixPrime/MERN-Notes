# Node & Express Backend
## Initialization
On terminal :
```
mkdir <project_name>
mkdir backend
cd backend
```
create module folder, package.json, etc.

` npm init -y `

` npm install -g nodemon`

## Editing created files
Go to -> package.json
add : "Type" : "module" to list @6th line

## Create Server.js

```js
import express from "express"
import cors from "cors"
import restaurants from "./api/restaurants.routes.js"

const app = express();

app.use(cors()); 
app.use(express.json());


app.use("/api/v1/restaurants",restaurants);
app.use("*",(req,res) => res.status(404).json({error: "not found"}));

export default app;
```
## Create .env
For setting global variables

Copy the DB URL from cloud.mongodb and replace the "myFirstDatabase" with database of your choice

```
RESTREVIEWS_DB_URI=mongodb+srv://m001-student:m001-mongodb-basics@sandbox.gjgom.mongodb.net/sample_restaurants?retryWrites=true&w=majority
RESTREVIEWS_NS=sample_restaurants
PORT=5000
```
## Create index.js
For connecting database to our application
```js
import app from "./server.js"
import mongodb from "mongodb"
import dotenv from "dotenv"

dotenv.config()
const MongoClient = mongodb.MongoClient;
const port = process.env.PORT || 8000;

//Connect to database
MongoClient.connect(
    process.env.RESTREVIEWS_DB_URI,
    {
        maxPoolSize:50, //Maximum 50 people can connect 
        wtimeoutMS:2500, // Request will expire after 2500 ms
        useNewUrlParser:true}
    )
    //Looking out for errors
    .catch(err => {
        console.error(err.stack);
        process.exit(1); // Exit
    })

    //Start the web server
    .then(async client =>{
        app.listen(port, ()=>{
            console.log(`listening on port ${port}`);
        })
    })

```
## Create APIs
Create restaurants.routes.js
```js
import express from "express"

const router = express.Router();

router.route("/").get((req,res)=>res.send("Wasuuuuup!")) // Demo route for "/"

export default router
```
Basic server is ready to run

on terminal do : 
`nodemon server`

Go to : localhost:5000/api/v1/restaurants

## Creating API for Data Object Access (DAO)
Create dao directory in API dir.

create : restaurantsDAO.js:
This part handles the query

* Establish Connection with Collection of DB
* Get query details and form a query
* Retrive the information based on query and return it

```js
let restaurants

export default class restaurantsDAO { //Establish Connection with collection
    static async inject(conn) {
        if (restaurants) {
            return;
        }
        try {
            restaurants = await conn.db(process.env.RESTREVIE_NS).collection("restaurant");
        } catch (e) {
            console.error(`Unable to establish connection handle in restaurantsDAO:${e}`);
        }
    }

    static async getRestaurants({ // Handle query details
        filters = null,
        page = 0,
        restaurantsPerPage = 20,
    } = {}) {
        let query
        if (filters) {
            if ("name" in filters) { //name of restaurant is specified
                query = { $text: { $search: filters["name"] } }
            } else if ("cuisine" in filters) { //cuisine is specified
                query = { "$cuisine": { $eq: filters["cuisine"] } }
            } else if ("zipcode" in filters) {
                query = { "address.zipcode": { $eq: filters["zipcode"] } }
            }
        }

        let cursor;
        try {
            cursor = await restaurants.find(query);
        } catch (e) {
            console.error(`Unable to issue find command ${e}`)
            return { restaurantsList: [], totalRestaurants: 0 }
        }

        const displayCursor = cursor.limit(restaurantsPerPage).skip(restaurantsPerPage * page)

        try {
            restaurantsList = await displayCursor.toArray(); //Restaurants List
            totalNumRestaurants = await restaurants.countDocuments(query); // Total Restaurants

            return { restaurantsList, totalNumRestaurants };
        } catch (e) {
            console.error(
                `Unable to convert cursor to array or problem counting documents : ${e}`
            )
            return { restaurantsList: [], totalNumRestaurants: 0 }
        }
    }
}
```
Import This DAO in index.js
```js
import RestaurantsDAO from "./api/dao/restaurantsDAO.js"
```
## Create Query Controller
create restaruants.controller.js in api dir





