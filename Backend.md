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

* Analyse the request and it's query parameters
* Set filters accordingly
* Call the DAO and get data
* Set the response  

```js
import RestaurantsDAO from "./dao/restaurantsDAO.js"

export default class RestaurantsController {
    static async apiGetRestaurants(req, res, next) {
        const restaurantsPerPage = req.query.restaurantsPerPage ? parseInt(req.query.restaurantsPerPage, 10) : 20
        const page = req.query.page ? parseInt(req.query.page, 10) : 0

        let filters = {}
        if (req.query.cuisine) {
            filters.cuisine = req.query.cuisine
        } else if (req.query.zipcode) {
            filters.zipcode = req.query.zipcode
        } else if (req.query.name) {
            filters.name = req.query.name
        }

        const { restaurantsList, totalNumRestaurants } = await RestaurantsDAO.getRestaurants({
            filters,
            page,
            restaurantsPerPage,
        })

        let response = {
            restaurants: restaurantsList,
            page: page,
            filters: filters,
            entries_per_page: restaurantsPerPage,
            total_results: totalNumRestaurants,
        }
        res.json(response)
    }
}
```
## Make New Changes to routes and index
Change Route to :
```js
import RestaurantsCtrl from "./restaurants.controller.js" 
router.route("/").get(RestaurantsCtrl.apiGetRestaurants); //Get Response as Restataurants
```

Before starting listening on port Get DB client :
```js
await RestaurantsDAO.injectDB(client);
    app.listen(port, () => {
        console.log(`listening on port ${port}`);
    }
```
## Creating Reviews Feature
* Add new route to restaurants.routes.js after get restaurants route
* Add all types of requests

### New restaurants.routes.js
```js
import express from "express"
import RestaurantsCtrl from "./restaurants.controller.js"
import ReviewsCtrl from "./reviews.controller.js"

const router = express.Router();

router.route("/").get(RestaurantsCtrl.apiGetRestaurants); //Get Response as Restataurants

router
    .route("/review")
    .post(ReviewsCtrl.apiPostReview)
    .put(ReviewsCtrl.apiUpdateReview)
    .delete(ReviewsCtrl.apiDeleteReview)

export default router
```

###  Create reviews.controller.js
*  inside it create ReviewsCtrl Class with methds like :
    * apiPostReview
    * apiUpdateReview
    * apiDeleteReview
```js
import ReviewsDAO from "./dao/reviewsDAO.js"

export default class ReviewsController {
    static async apiPostReview(req, res, next) {
        try {
            const restaurantId = req.body.restaurant_id
            const review = req.body.text
            const userInfo = {
                name: req.body.name,
                _id: req.body.user_id
            }
            const date = new Date()

            const ReviewResponse = await ReviewsDAO.addReview(
                restaurantId,
                userInfo,
                review,
                date,
            )
            res.json({ status: "success" })
        } catch (e) {
            res.status(500).json({ error: e.message })
        }
    }

    static async apiUpdateReview(req, res, next) {
        try {
            const reviewId = req.body.review_id
            const text = req.body.text
            const date = new Date()

            const reviewResponse = await ReviewsDAO.updateReview(
                reviewId,
                req.body.user_id,
                text,
                date,
            )

            var { error } = reviewResponse
            if (error) {
                res.status(400).json({ error })
            }

            if (reviewResponse.modifiedCount === 0) {
                throw new Error(
                    "unable to update review - user may not be original poster",
                )
            }

            res.json({ status: "success" })
        } catch (e) {
            res.status(500).json({ error: e.message })
        }
    }

    static async apiDeleteReview(req, res, next) {
        try {
            const reviewId = req.query.id
            const userId = req.body.user_id
            console.log(reviewId)
            const reviewResponse = await ReviewsDAO.deleteReview(
                reviewId,
                userId,
            )
            res.json({ status: "success" })
        } catch (e) {
            res.status(500).json({ error: e.message })
        }
    }

}
```
    
### Create reviewsDAO.js
Create Class ReviewsDAO with following methods:
* injectDB(conn)
* addReview
* updateReview
* deleteReview
```js
import mongodb from "mongodb"
const ObjectId = mongodb.ObjectId

let reviews

export default class ReviewsDAO {
    static async inject(conn) {
        if (reviews) {
            return
        }

        try {
            reviews = await conn.db(process.env.RESTREVIEWS_NS).collection("reviews")
        } catch (e) {
            console.error(`Unable to find collection handles in user DAO :${e}`)
        }
    }

    static async addReview(restaurantID, user, review, date) {
        try {
            const reviewDoc = {
                user_id: user._id,
                date: date,
                text: review,
                restaurant_id: ObjectId(restaurantID)
            }
            return await reviews.insertOne(reviewDoc)
        } catch (e) {
            console.error(`Unable to post review${e}`)
            return { error: e }
        }
    }

    static async updateReview(reviewId, userId, text, date) {
        try {
            const updateResponse = await reviews.updateOne({ user_id: userId, _id: ObjectId(reviewId) }, { $set: { text: text, date: date } })
            return updateResponse;
        } catch (e) {
            console.error(`Unable to update the review:${e}`)
            return { error: e }
        }
    }

    static async deleteReview(reviewId, userId) {
        try {
            const deletResponse = await reviews.deleteOne({
                _id: ObjectId(reviewId),
                user_id: userId
            })
        } catch (e) {
            console.error(`Unable to delete review : ${e}`)
            return { error: e }
        }
    }
}
```
Now
## Add Feature of Getting Cuidines and Specific Restaurans
### Add to restaurant.routes.js :
```js
router.route("/id/:id").get(RestaurantsCtrl.apiGetRestaurantsById);
router.route("cuisines").get(RestaurantsCtrl.apiGetRestaurantsByCuisines)
```
### Adding These New Methods to RestaurantsCtrl
Go to restaurants.controller.js and inside RestaurantsCtrl Class Add :
```
static async apiGetRestaurantById(req, res, next) {
    try {
      let id = req.params.id || {}
      let restaurant = await RestaurantsDAO.getRestaurantByID(id)
      if (!restaurant) {
        res.status(404).json({ error: "Not found" })
        return
      }
      res.json(restaurant)
    } catch (e) {
      console.log(`api, ${e}`)
      res.status(500).json({ error: e })
    }
  }

  static async apiGetRestaurantCuisines(req, res, next) {
    try {
      let cuisines = await RestaurantsDAO.getCuisines()
      res.json(cuisines)
    } catch (e) {
      console.log(`api, ${e}`)
      res.status(500).json({ error: e })
    }
  }
```
### Implement these methods in restaurantsDAO.js 
New restaurantsDAO.js will be : 
```js
import mongodb from "mongodb"
const ObjectId = mongodb.ObjectId
let restaurants

export default class RestaurantsDAO {
  static async injectDB(conn) {
    if (restaurants) {
      return
    }
    try {
      restaurants = await conn.db(process.env.RESTREVIEWS_NS).collection("restaurants")
    } catch (e) {
      console.error(
        `Unable to establish a collection handle in restaurantsDAO: ${e}`,
      )
    }
  }

  static async getRestaurants({
    filters = null,
    page = 0,
    restaurantsPerPage = 20,
  } = {}) {
    let query
    if (filters) {
      if ("name" in filters) {
        query = { $text: { $search: filters["name"] } }
      } else if ("cuisine" in filters) {
        query = { "cuisine": { $eq: filters["cuisine"] } }
      } else if ("zipcode" in filters) {
        query = { "address.zipcode": { $eq: filters["zipcode"] } }
      }
    }

    let cursor
    
    try {
      cursor = await restaurants
        .find(query)
    } catch (e) {
      console.error(`Unable to issue find command, ${e}`)
      return { restaurantsList: [], totalNumRestaurants: 0 }
    }

    const displayCursor = cursor.limit(restaurantsPerPage).skip(restaurantsPerPage * page)

    try {
      const restaurantsList = await displayCursor.toArray()
      const totalNumRestaurants = await restaurants.countDocuments(query)

      return { restaurantsList, totalNumRestaurants }
    } catch (e) {
      console.error(
        `Unable to convert cursor to array or problem counting documents, ${e}`,
      )
      return { restaurantsList: [], totalNumRestaurants: 0 }
    }
  }
  static async getRestaurantByID(id) {
    try {
      const pipeline = [
        {
            $match: {
                _id: new ObjectId(id),
            },
        },
              {
                  $lookup: {
                      from: "reviews",
                      let: {
                          id: "$_id",
                      },
                      pipeline: [
                          {
                              $match: {
                                  $expr: {
                                      $eq: ["$restaurant_id", "$$id"],
                                  },
                              },
                          },
                          {
                              $sort: {
                                  date: -1,
                              },
                          },
                      ],
                      as: "reviews",
                  },
              },
              {
                  $addFields: {
                      reviews: "$reviews",
                  },
              },
          ]
      return await restaurants.aggregate(pipeline).next()
    } catch (e) {
      console.error(`Something went wrong in getRestaurantByID: ${e}`)
      throw e
    }
  }

  static async getCuisines() {
    let cuisines = []
    try {
      cuisines = await restaurants.distinct("cuisine")
      return cuisines
    } catch (e) {
      console.error(`Unable to get cuisines, ${e}`)
      return cuisines
    }
  }
}
```
That's it !




