# Building Dynamic Filter Queries with the Harvard Art Museums API: Structuring Efficient Backend Queries

In the CurateSphere application, managing filters dynamically was a critical part of enhancing the user experience when browsing through artworks. The Harvard Art Museums API provides rich filtering capabilities, allowing us to retrieve objects by various criteria, such as `medium`, `culture`, and `period`. However, the challenge was to construct dynamic queries from user inputs and format them correctly to match the API’s expected structure.

### Structuring the Backend Query for Filters

To achieve dynamic querying, we built a backend function that took the user's filter selections and transformed them into a valid query string. The Harvard API expects filter parameters in the form of query strings, and our job was to handle that dynamically. Let’s walk through how we did it.

For example, a request to the Harvard API might look like this:

```bash
https://api.harvardartmuseums.org/object?apikey=${API_KEY}&medium=oil
```

But with our application’s UI, users can select multiple filters at once, such as oil or acrylic medium and American culture. This would translate to:

```bash
https://api.harvardartmuseums.org/object?apikey=${API_KEY}&medium=oil|acrylic&culture=american
```

The goal was to transform user-selected filter objects into this format, where multiple filter values are separated by the pipe (`|`) character. Here's how we approached that in the frontend service file then in the backend controller.


### Transforming the Filter Object into a Query String

#### Frontend Service File
The object comes into this util function as `{size:12, {medium: {oil:2028390, acrylic:54321}, culture: {american:123}}`
```js
// creates query string out of an object
const getQueryString = (query) => {
// start with the size, which is always included
  let finalStr = `&size=${query?.size}`;
  if (!query) return;
  // this logic looks for nested objects and converts them to a pipe str if there are more than 1 value associated with it
  for (let primaryCategory in query) {
    if (typeof query[primaryCategory] === "object") {
      finalStr =
        finalStr +
        `&${primaryCategory}=` +
        Object.values(query[primaryCategory]).join("|");
    }
  }

  return finalStr;
};
```
The above function will take the values of the categories in our nested object and send that along with the primary category name, but now it will include all of the values separated by the "|", which is aboslutely necessary for the Harvard API to combine filter values correctly. This is used as a helper to the following function which communicates with our backend. These are sent as a request query to our backend. 

```js
//////////////////////////////////////////////////////
// GET | Get All Artworks With Filter
//////////////////////////////////////////////////////
export const getAllArtworks = async (filters) => {
  // this allows us to ensure the string will start as a query with a ? and all following
  // filters will follow a pattern of &filterKey=value
  const query = getQueryString(filters);
  const startQuery = "?" + query.slice(1);
  try {
    const response = await fetch(`${BACKEND_URL}/artworks/search${startQuery}`);
    const data = await response.json();
    if (response.ok) {
      return data;
    } else {
      throw new Error();
    }
  } catch (err) {
    console.error(err);
    console.log(`Unable to communicate with DB to get all artworks`);
  }
};
```

On the backend, we used a helper function to dynamically convert the req.query into a format useable in our call to the Harvard API. Below is the code that handled this transformation:
Before this helper is ran, the value of req.query is `{ size: '12', medium: '2028390|2035850', culture:'123' }`
```js
// Transform object to query string
const getQueryString = (query) => {
  if (!query) return;
  return Object.keys(query)
    .map((key) => `&${key}=${query[key]}`)
    .join("");
};
```
So now this req.query converts into a query string that can be appended to our API request URL. The `Object.keys()` method loops through each filter and transforms it into the required format. 

Next, we applied this function in our `getArtworks` API controller:

```js
///////////////////////////
// GET | Get artworks based on filters
///////////////////////////
const getArtworks = async (req, res) => {
  // Convert the filter object into a query string for the Harvard API
  const queriedFilter = getQueryString(req.query);

  try {
    const response = await fetch(
      `${BASE_URL}/object?apikey=${API_KEY}${queriedFilter}`
    );
    const data = await response.json();

    // Update API key placeholders
    data.info.next = swapApiKeyAndPlaceholder(data.info.next, "API_KEY");
    data.info.prev = "";

    res.status(200).json(data);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "cannot get all artworks" });
  }
};
```

Here’s what happens in this function:
1. We use the `getQueryString()` helper to take the filters from `req.query` (sent from the frontend) and transform them into a query string.
2. The query string is then appended to the Harvard API URL.
3. A request is made to fetch the artworks that match the selected filters.
4. The response from the Harvard API is processed and returned to the frontend.

### Handling the Filter State in the Frontend

On the frontend, we maintain the selected filters in the component’s state. When the user interacts with the UI (like selecting checkboxes for different filters), we update this state, and when they submit their selection, we send the state as a query object to the backend.

Here’s an example of how we construct the filter state:

```js
const filterState = {
  size: 12, 
  medium: { oil: 12345 }
};
```
### Recap
	1.	Understanding the API’s Requirements
	2.	Transforming the Filter State
	3.	Backend Query Construction
	4.	Fetching and Returning the Data
