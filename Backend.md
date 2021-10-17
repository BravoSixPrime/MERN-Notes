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
import restaurants from "./api/restaurants.route.js"

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
RESTREVIEWS_DB_URI = mongodb+srv://m001-student:m001-mongodb-basics@sandbox.gjgom.mongodb.net/sample_restaurants?retryWrites=true&w=majority
RESTREVIEWS_NS = sample_restaurants
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
    proces.env.RESTREVIWS_DB_URI,
    {
        poolSize:50, //Maximum 50 people can connect 
        wtimeout:2500, // Request will expire after 2500 ms
        newUrlParser:true
    }
    //Looking out for errors
    .catch(err=>{
        console.error(err.stack);
        process.exit(1); // Exit
    })
    
    //Start the web server
    .then(async client =>{
        app.listen(port,()=>{
            console.log(`listening on port ${PORT}`);
        })
    })
)
```



