# Connecting the backend

We rename the starter-code folder to client because this is our React app. Then we add another folder on the root level that we call server.

```bash
$ mkdir server
```

```bash
$ cd server
$ touch app.js
$ npm init 
$npm install express
```

First we create the basic server setup with Express and import the countries.json file.

```js
// server/app.js
const express = require('express');
const app = express();

const countries = require('../client/src/countries.json');

app.get('/countries', (req, res) => {
    res.json(countries);
});

app.listen(5555, () => {
    console.log('Server listening on Port 5555');
});
```

If we now run the server and visit http://localhost:5555/countries we should see the JSON response

```bash
$ node server/app.js
```

## Client

Now let's update the code in the client to use our own server instead of the JSON file.

In the client folder run 
```bash
$ npm install axios
```

In the App.js we now want to fetch the countries from the server
We need to add state to the component. In the componentDidMount() lifecycle method we make a request to our server.
If we try the URL 'http://localhost:5555/countries we have an issue with CORS

https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS

But we can add a proxy to our React app - then if we use the URL '/countries' in the GET request then this translates from 'localhost:3000/countries' to 'localhost:5555/countries'

```js
// package.json
//
"name": "wiki-countries",
"version": "0.1.0",
"proxy": "http://localhost:5555",
"private": true,
//
```

```js
// src/App.js
import axios from 'axios';

class App extends React.Component {

  state = {
    countries: []
  }

  componentDidMount() {
    axios.get('/countries')
      .then(response => {
        console.log(response);
      })
  }
```

Then we want to add the countries to the state and then we pass them to the CountriesList as a prop.

```js
//
componentDidMount() {
  axios.get('/countries')
    .then(response => {
      console.log(response);
      this.setState({
        countries: response.data
      })
    })
}
//
<div className='row'>
  <CountryList countries={this.state.countries} />
  <Route exact path='/:id' component={CountryDetail} />
</div>
//
```

In the CountryList.js component we now use the prop instead of the imported JSON file.

```js
// src/components/CountryList.js
//
const CountryList = (props) => {
  return (
    <div className="col-5" style={{ maxHeight: '90vh', overflow: 'scroll' }}>
      <div className='list-group'>
        {props.countries.map((country, index) => {
          return (
//
```
Now we want to also use our server in the CountryDetail.js component.

At the moment we are filtering the country that should be displayed from the countries JSON file. 

Instead we want to get the data for one specific country from our server. 

For that we have to add another route to our server first.

The borders array contains only the cca3 ids. So we want to replace them with the complete country objects. Because otherwise we would have to make in the client another request for every border country when we render the borders.

We also make a copy of the object to not modify the original.

```js
// server/app.js
//
const getCountryByCode = cca3 => countries.find(el => el.cca3 === cca3);

app.get('/countries/:countryCode', (req, res) => {
    const country = { ...getCountryByCode(req.params.countryCode) };
    country.borders = country.borders.map(cca3 => getCountryByCode(cca3));

    res.json(country);
});
//
```

Now in the CountryDetail.js component we add the state and a get request to the server.

The country in the state we set to the initial value of null.

```js
// src/components/CountryDetail.js
//
import axios from 'axios';


class CountryDetail extends React.Component {
  // we add the state 
  state = {
    country: null
  };

  // we add a function to get the data from the server
  getCountryData = () => {
    const countryCode = this.props.match.params.id;
    axios.get(`/countries/${countryCode}`).then(response => {
      const country = response.data;
      console.log(response.data);
      this.setState({
        country: country
      });
    });
  };

  // when the component got mounted
  componentDidMount() {
    this.getCountryData();
  }



  render() {
    // here we reference the country from the state now
    const country = this.state.country;

    if (!country) return <></>;
    return (
      <div className="col-7">
        <h1>{country.name.common}</h1>
        <table className="table">
          <thead></thead>
          <tbody>
            <tr>
              <td style={{ width: "30%" }}>Capital</td>
              <td>{country.capital}</td>
            </tr>
            <tr>
              <td>Area</td>
              <td>
                {country.area} km
                <sup>2</sup>
              </td>
            </tr>
            {country.borders.length > 0 && (
              <tr>
                <td>Borders</td>
                <td>
                  <ul>
                    {country.borders.map(el => {
                      return (
                        <li key={el.cca3}>
                          <Link to={`/${el.cca3}`}>
                            {el.name.common}
                          </Link>
                        </li>
                      );
                    })}
                  </ul>
                </td>
              </tr>
            )}
          </tbody>
        </table>
      </div>
    );
  }
}

export default CountryDetail;
```

Now we have a problem: When we click on one of the border countries the the data gets not updated because the getData method is only triggered at componentDidMount. So we have to find a way to check if the country in the state is not the same as the country that's cca3 is in the URL -> that got clicked.

To do that we use the lifecycle method componentDidUpdate() where we have access to the previous props.

```js
// src/components/CountryDetail.js
//
componentDidUpdate(prevProps) {
  console.log('previous props: ', prevProps);
  console.log('current props: ', this.props);
  // if (prevProps.match.params.cca3 !== this.props.match.params.cca3) {
  if (prevProps !== this.props) {
    this.getCountryData();
  }
}
//
```

If we now change the initial value of the country in the state to {} then we get an error -> country.name is not defined.

The problem is that the check in the render method that is checking for state.country to be false does not work with empty object. 

So then the rendering of the country.name etc is tried on {} and that results in an error. That's what the check on line 40 in CountryDetail.js is for -> to make sure that we don't render without having the correct data in the state. 

This illustrates that the render method gets executed before the componentDidMount(). When the render get's executed the first time the fetching of the data did not take place yet because that is only triggered when componentDidMount() gets executed. 