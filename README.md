# Show Rules applied per Application using Auth0 Management API v2

This is a sample application that will list down the Rules running on each Auth0 Client. On successful login, the 
application page will show the list of all Clients and the Rules that run on each of them. A sample deployment looks as 
follows:
![App screenshot](/doc/app-screenshot.png?raw=true)

Please follow the following steps to setup the application code on your localhost.

## Auth0 configuration
1. You need to first create a Node.js client in Auth0. Go to the [Auth0 Clients page](https://manage.auth0.com/#/clients) and click on `Create Client` button. Then select `Regular Web Applications` and click on the Create 
button. Select `Node.js` from the list of available technologies. Give a name to the client, e.g. `ListAppRulesUsingMngmtApiV2`

2. Add `http://localhost:3000/callback` to the list of Allowed Callback URLs in the client settings page.

3. Create a Non Interactive Client `API Explorer Client`. We will need this client to make calls to the Management API 
from our application code.

4. Create a Whitelist Rule with following code
```javascript
function (user, context, callback) {
    if (context.clientName === 'ListAppRulesUsingMngmtApiV2') {
      var whitelist = [ 'youremail@example.com' ]; //authorized users
      var userHasAccess = whitelist.some(
        function (email) {
          return email === user.email;
        });

      if (!userHasAccess) {
        return callback(new UnauthorizedError('Access denied.'));
      }
    }
    callback(null, user, context);
}
```
This way your application can be accessed only by a list of authorized users


## Running the Sample

1. Download or git clone this code to your localhost.

```bash
git clone git@github.com:souvikbasu/auth0-rules-per-app.git
cd auth0-rules-per-app
```
2. Install the dependencies.

```bash
npm install
```

3. Configure environment variables
```bash
# copy configuration and replace with your own
cp .env.example .env
```

`.env` file contains your Auth0 Client ID and Secret. Replace 
the values for `AUTH0_CLIENT_ID`, `AUTH0_DOMAIN`, `AUTH0_CLIENT_SECRET`, `MANAGEMENT_API_CLIENT_ID` and `MANAGEMENT_API_CLIENT_SECRET` with your Auth0 credentials. If 
you don't  yet have an Auth0 account, [sign up](https://auth0.com/signup) for free.

Following are the keys set in the .env file

`AUTH0_CLIENT_ID` Get the Client ID from the [Auth0 Clients page](https://manage.auth0.com/#/clients)

`AUTH0_DOMAIN` Click on this Client in the [Auth0 Clients page](https://manage.auth0.com/#/clients) and get the 
Domain 

`AUTH0_CLIENT_SECRET` Click on this client in the [Auth0 Clients page](https://manage.auth0.com/#/clients) and get 
the Client Secret

`MANAGEMENT_API_CLIENT_ID` Click on *API Explorer Client* in the [Auth0 Clients page](https://manage.auth0.com/#/clients) and get the Client ID. You need to create an API Explorer Client if it does not already exist. This 
client is needed to query the Management API

`MANAGEMENT_API_CLIENT_SECRET`  Click on *API Explorer Client* in the [Auth0 Clients page](https://manage.auth0.com/#/clients) and get the Client Secret
 
`AUTH0_CALLBACK_URL` If you are running the application on localhost then you do not need to set this key. If you 
deploy the application to a third party service provider like Heroku then you need to specify the value as 
`AUTH0_DOMAIN`/callback. You will need to add this url to Allowed Callback URLs list in the Client's settings page in Auth0



5. Run the app.

```bash
npm start
```

The app will be served at `localhost:3000`. To deploy the application to Heroku create an account in https://www.heroku.com/. Then commit the application source files to heroku repo. I recommend using the [heroku cli](https://devcenter.heroku.com/articles/heroku-cli) Add the 
settings from .env file to 
https://dashboard.heroku.com/apps/your-app-name/settings

You might choose to deploy your app with any other service provider as per your comfort with their service.


## Explanation of code logic
The application code uses Auth0 lock to authenticate the user. The Whitelist rule makes sure that only authorized 
users have access to this application.
 
If the user is not authorized the user is redirected to page /notAuthorized which shows the error message to the user.
Otherwise the user is shown the list of applications and all rules that apply to each application. Following 
conditions are handled to show the rules per application:
* Show list of Rules which apply to the application using condition like `if (context.clientName === 'App Name') {`
* Show list of Rules which apply to the application using condition like `if (context.clientID === 
'VxYJCEfNONlpSZVAuD4uRKGKpz8abcda') {`
* Do NOT show a rule for an application if condition is negative like `if (context.clientName !== 'App Name') {` or 
`if (context.clientID !== 'VxYJCEfNONlpSZVAuD4uRKGKpz8Jm6mh') {`
* Show an application in list even if no rules apply to that application
 
We find out which application a rule applies (or does not apply) by looking at the Rules script and using string 
Regular Expressions to parse it.

There are two preferable algorithms to achieve this. Following are the pusedo code. We have used the code which 
executes faster:
### More readable code
Since the problem statement is around rules per client, a readable code will be to loop through each client and find out all matching rules for the client

```
Get list of all rules and store in `rules` variable
Get list of all clients and store in `clients` variable

For client in clients:
    For rule in rules:
        If rule.script has text like `if (context.clientName === '{ client name }')` or `if (context.clientID === '{ client id }')`
        Then: 
             Add rule to client.rules
            
        If rule.script has text like `if (context.clientName !== '{ client name }')` or `if (context.clientID !== '{ client id }')`
        Then: 
             clientDisallowed = Find out client from clients that this rule mentions 
             If client != clientDisallowed
             Then:
                Add this rule to array of client.rules
             
client.rules in all clients contains the list of rules that apply to this client                    
```
 
The above algorithm runs the regex on rules script 4 times for each client which makes the code slower. 
Hence I have chosen to go with a faster algorithm that matches regex against each rule script only 4 times 
irrespective of the client. The regex is not matched again and again for each client.
 
### Faster code
```
Get list of all rules and store in `rules` variable
Get list of all clients and store in `clients` variable

For rule in rules:
    If rule.script has text like `if (context.clientName === '{ client name }')` or `if (context.clientID === '{ client id }')`
    Then: 
         clientAllowed = Find out client from clients that this rule mentions 
         Add rule to clientAllowed.rules
        
    If rule.script has text like `if (context.clientName !== '{ client name }')` or `if (context.clientID !== '{ client id }')`
    Then: 
         clientDisallowed = Find out client from clients that this rule mentions 
         For all other clients, Add this rule to array of client.rules
             
client.rules in all clients contains the list of rules that apply to this client                    
```
 

## Sample Deployment
You can check a sample [Heroku](https://auth0-rules-per-app.herokuapp.com) deployment of this application if your 
email has been added in the Whitelist users for this application.

## License

This project is licensed under the MIT license. See the [LICENSE](LICENSE) file for more info.
