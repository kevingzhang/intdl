# System design using Meteor.js
This system design is based on my current limited knowledge about Intelligent Dealer project.
It is in draft mode, will be update at any time.

In this document, I will describe the design goal, technical benefits, risks on each part.

## Modular using Templates

I got a few design pictures. base on those pictures, I have a brief layout feeling of this project.

### Design goal
Meteor.js has a enhanced version of Handlebar.js templating system, called Spacebar. Using Spacebar, we have a lot of benefits
- Separate each block of UI into different template. The logic (MVC) are inside the template. They can be design, code, test independently. For example, we can release *alert* feature first, then add other features when ready.
- Each block can be turn on/off according to user's privilege. For example, when we have the licensing system, a user can have some of features, other features are hidden until he pay for those advanced features.

### Technical detail

**TBD**

## Data flow and multiple cache layer

### Design goal
- Separate backend logic with front end presentation. Clarence and I can work on different layer without interference.
- Reactive data source. As long as the data changed at the backend server, the UI data will change automatically in real time.
- Fastest user interactive response. User can interactive with data on the UI without ajax round trip to the server to get data when possible, so that no latency can be noticed.
- If hitting server for data is inevitable, try to get data from Meteor's server data, without hitting the backend server for data aggregation, when possible.
- If we have to hit the backend system to get aggregated data, We will cache in Meteor's server database, so that later access can be returned without hit backend again

### Technical detail

We have 3 layers of data cache.

#### Local mini-Mongo in Browser's memory

##### Cache and latency free interactive
This is the fastest cache layer, the most closest to user's UI. These data is a subset of Meteor's server side MongoDB dataset.
Because the data is store inside Browser's memory database, when user interact with the UI (such as change sort, filter, turn to next page...) the data can be retrieved from the memory database and display on the web page in no time.
There won't be any traffic between browser and server. No latency needed. User can see the data instantly.
##### Realtime data update via DDP
Client side miniMongo dataset and server site MongoDB syncup using DDP, with subscribe/publish mechanism.

When data changed in server site mongoDB, the DDP will automatically update the same data in client site mini Mongo, as well as the data displayed in UI. This happens in real time.
##### When hit missing in this cache layer, refresh is needed
When the data not found in miniMongo, or change of subscribe condition, a new refresh will happen. That means new data will published from server mongo to client site miniMongo.

#### Meteor's server side MongoDB
This is the second cache layer. This layer is in between the client side miniMongo and backend server. The existing of this layer can minimize the hit to backend server. As long as the required data was previously cached, it will be send back to client directly, without hit the backend system.
##### Low latency
The data in MongoDB is the full set of aggregated table data. (Compare to the client side miniMongo only store a subset).
Client miniMongo only have a subset data because minimize the use of browser's memory. So when user change some query parameter, for example, change to another dealer, or change to a different date range, the client side miniMongo doesn't have the data stored, we need to publish/subscribe the data sync down to client from server side mongoDB

Because the server side mongoDB stores aggregated data (the same format as data table), this process is very fast. Although it is not 0 latency, but it is very small latency, *almost* instant response

##### Missing hits and cache

If the this layer doesn't have the query data, for example, a new dealer, new product, or a date range which never been queried before, the server mongoDB won't have the data. In this case, we need to hit the backend server

When query the backend server, we also save the data in MongoDB, so next time the same query comes from the client, we do not need to hit backend server again.

##### Logic to detect out-of-date data

An important logic here is how we can determine if the server side mongoDB data is still up-to-date.
If we simply return any data to client as long as we find the data match the query paramter, we may be ended with an out-of-date data shown to the user.
We need algorithm to find out if the data *could be* expired. If there is such possiblity, we do the following two steps
- Send the current data to client first. So the user will see the *possible* out-of-date data first. It is better to show user some *possible* old data then an empty table.
- At the same time, in front end, user can see a "refreshing" icon on the UI, indicating that we are trying to get latest data for you. In backend, we hit the backend server to get latest data. Then, compare the latest data with current stored data. If they are identical, then do nothing. Because the data is up-to-date. If the data is not identical, then update the mongoDB (Remember, the DDP can sync up the client in realtime) so that the user can see updated data shortly after original expired data.

We can also log the hit record, so that we can statistic later to find out which query is never expired, so that we can modify out logic to not refresh it in the future.


#### Backend server (Clarence's work)

This is the last data source, This is not a cache. When data query comes to this layer, that means this query is never happened before, or expired already. This is last catch to get latest data from our backend system.
Clarence can package the backend data layer into a common.js package, exports some query interface function to me. The input would be query parameters, out put would be JSON formatted table data, which I can store into system mongoDB directly.

###

## Realtime alert

Due to the realtime DDP featured by Meteor. Realtime alert is very easy. As long as the system need to show an alert, just put an alert object into server side MongoDB. The UI will display the alert in no time.


## URL parsing and status snapshots

**TBD**

## Testing

**TBD**

## Development schedule

### Phrase one: Product design and technical mockup
#### Product design
UI design, all the details, interactive design.
#### Technical mockup
- Mockup on the layout,
- data syncup,
- communication between Meteor and backend service.
- Create some testing data. Will be used to verify, or future unit test data
- No UI detail need to be concern at this moment

### Phrase two: Pilot view
Use one view (for example, a typical alert view ) as a pilot. Implement most features of this view only.
- Test the mockup, try to figure out all potential problem, performance evluation
- Test the development speed. Estimate the overall implementation cost of all views
- Show some pilot users, get feedback

### Phrase three: Implement all views one by one. From high priority to low.
- If the market need some urgent views to be completed first, we can go those first. Once ready, we can start sale while development on other views keep going
- Once a view is done, go to testing procedure. When testing done, go to documentation procedure, then go to market

### Phrase four: Testing and improvements.
