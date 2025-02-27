# How to build an agent with the Node.js SDK and function-calling

OpenAI functions enable your app to take action based on user inputs. This means that it can, e.g., search the web, send emails, or book tickets on behalf of your users, making it more powerful than a regular chatbot.

In this tutorial, you will build an app that uses OpenAI functions along with the latest version of the Node.js SDK. The app runs in the browser, so you only need a code editor and, e.g., VS Code Live Server to follow along locally. Alternatively, write your code directly in the browser via [this code playground at Scrimba.](https://scrimba.com/scrim/c6r3LkU9)

If you prefer watching screencasts over reading, then you can check out [this scrim, which walks through the code line-by-line:](https://scrimba.com/scrim/co0044b2d9b9b7f5bf16e0391)

<iframe src="https://scrimba.com/scrim/co0044b2d9b9b7f5bf16e0391?embed=openai,no-header" width="100%" height="500"></iframe>

## What you will build

Our app is a simple agent that helps you find activities in your area.
It has access to two functions, `getLocation()` and `getCurrentWeather()`,
which means it can figure out where you’re located and what the weather
is at the moment. 

At this point, it's important to understand that
OpenAI doesn't execute any code for you. It just tells your app which
functions it should use in a given scenario, and then leaves it up to
your app to invoke them.

Once our agent knows your location and the weather, it'll use GPT’s
internal knowledge to suggest suitable local activities for you.

## Importing the SDK and authenticating with OpenAI

We start by importing the OpenAI SDK at the top of our JavaScript file and authenticate with our API key, which we have stored as an environment variable.

``` js
import OpenAI from "openai";

const openai = new OpenAI({
    apiKey: process.env.OPENAI_API_KEY,
    dangerouslyAllowBrowser: true
});
```

Since we're running our code in a browser environment at Scrimba, we also need to set `dangerouslyAllowBrowser: true` to confirm we understand the risks involved with client-side API requests. Please note that you should move these requests over to a Node server in a production app.

## Creating our two functions

Next, we'll create the two functions. The first one - `getLocation` -
uses the [IP API](https://ipapi.co/) to get the location of the
user.

``` js
async function getLocation() {
    const response = await fetch('https://ipapi.co/json/');
    const locationData = await response.json();
    return locationData;
}
```

The IP API returns a bunch of data about your location, including your
latitude and longitude, which we’ll use as arguments in the second
function `getCurrentWeather`. It uses the [Open Meteo
API](https://open-meteo.com/) to get the current weather data, like
this:

``` js
async function getCurrentWeather(latitude, longitude) {
    const url = `https://api.open-meteo.com/v1/forecast?latitude=${latitude}&longitude=${longitude}&hourly=apparent_temperature`;
    const response = await fetch(url);
    const weatherData = await response.json();
    return weatherData;
}
```

## Describing our functions for OpenAI

For OpenAI to understand the purpose of these functions, we need to
describe them using a specific schema. We'll create an array called
`functionDefinitions` that contains one object per function. Each object
will have three keys: `name`, `description`, and `parameters`.

``` js
const functionDefinitions = [
    {
        name: "getCurrentWeather",
        description: "Get the current weather in a given location",
        parameters: {
            type: "object",
            properties: {
                longitude: {
                    type: "string",
                },
                latitude: {
                    type: "string",
                }
            },
            required: ["longitude", "latitude"]
        }
    },
    {
        name: "getLocation",
        description: "Get the user's location based on their IP address",
        parameters: {
            type: "object",
            properties: {}
        }
    }
]
```

## Setting up the messages array

We also need to define a `messages` array. This will keep track of all of the messages back and forth between our app and OpenAI. 

The first object in the array should always have the `role` property set to `"system"`, which tells OpenAI that this is how we want it to behave.
``` js
const messages = [{
    role: "system",
    content: "You are a helpful assistant. Only use the functions you have been provided with."
}];

```


## Creating the agent function

We are now ready to build the logic of our app, which lives in the
`agent` function. It is asynchronous and takes one argument: the
`userInput`.

We start by pushing the `userInput` to the messages array. This time, we set the `role` to `"user"`, so that OpenAI knows that this is the input from the user.

``` js
async function agent(userInput) {
    messages.push([{
      role: "user",
      content: userInput,
    }]);
    const response = await openai.chat.completions.create({
        model: "gpt-4",
      messages: messages,
      functions: functionDefinitions
    });
    console.log(response);
}
```

Next, we'll send a request to the Chat completions endpoint via the
`chat.completions.create()` method in the Node SDK. This method takes a
configuration object as an argument. In it, we'll specify three
properties:

-   `model` - Decides which AI model we want to use (in our case,
    GPT-4).
-   `messages` - The entire history of messages between the user and the
    AI up until this point.
-   `functions` - A description of the functions our app has access to.
    Here, we'll we use the `functionDefinitions` array we created
    earlier.


## Running our app with a simple input

Let's try to run the `agent` with an input that requires a function call to give a suitable reply.

``` js
agent("Where am I located right now?");
```

When we run the code above, we see the response from OpenAI logged out
to the console like this:

``` js
{
    id: "chatcmpl-84ojoEJtyGnR6jRHK2Dl4zTtwsa7O",
    object: "chat.completion", 
    created: 1696159040, 
    model: "gpt-4-0613", 
    choices: [{
        index: 0, 
        message: {
            role: "assistant", 
            content: null, 
            function_call: {
                name: "getLocation", // The function OpenAI wants us to call
                arguments: "{}"
            }
        }, 
        finish_reason: "function_call" // OpenAI wants us to call a function
    }],
    usage: {
        prompt_tokens: 134, 
        completion_tokens: 6, 
        total_tokens: 140
    }
}
```

This response tells us that we should call one of our functions, as it contains the following key: `finish:_reason: "function_call"`.

The name of the function can be found in the
`response.choices[0].message.function_call.name` key, which is set to
`"getLocation"`.

## Turning the OpenAI response into a function call

Now that we have the name of the function as a string, we'll need to
translate that into a function call. To help us with that, we'll gather
both of our functions in an object called `availableFunctions`:

``` js
const availableFunctions = {
    getCurrentWeather,
    getLocation
};
```

This is handy because we'll be able to access the `getLocation` function
via bracket notation and the string we got back from OpenAI, like this:
`availableFunctions["getLocation"]`.

``` js

const { finish_reason, message } = response.choices[0];

if (finish_reason === "function_call") {
    const functionName = message.function_call.name;           
    const functionToCall = availableFunctions[functionName];
    const functionArgs = JSON.parse(message.function_call.arguments);
    const functionArgsArr = Object.values(functionArgs);
    const functionResponse = await functionToCall.apply(null, functionArgsArr); 
    console.log(functionResponse);
}
```

We're also grabbing ahold of any arguments OpenAI wants us to pass into
the function: `message.function_call.arguments`.
However, we won't need any arguments for this first function call.


If we run the code again with the same input
(`"Where am I located right now?"`), we'll see that `functionResponse`
is an object filled with location about where the user is located right
now. In my case, that is Oslo, Norway.

``` js
{ip: "193.212.60.170", network: "193.212.60.0/23", version: "IPv4", city: "Oslo", region: "Oslo County", region_code: "03", country: "NO", country_name: "Norway", country_code: "NO", country_code_iso3: "NOR", country_capital: "Oslo", country_tld: ".no", continent_code: "EU", in_eu: false, postal: "0026", latitude: 59.955, longitude: 10.859, timezone: "Europe/Oslo", utc_offset: "+0200", country_calling_code: "+47", currency: "NOK", currency_name: "Krone", languages: "no,nb,nn,se,fi", country_area: 324220, country_population: 5314336, asn: "AS2119", org: "Telenor Norge AS"}
```

We'll add this data to a new item in the `messages` array, where we also
specify the name of the function we called.

``` js
messages.push({
  role: "function",
  name: functionName,
  content: `The result of the last function was this: ${JSON.stringify(functionResponse)}
  `
});
```

Notice that the `role` is set to `"function"`. This tells OpenAI
that the `content` parameter contains the result of the function call
and not the input from the user.

At this point, we need to send a new request to OpenAI with this updated
`messages` array. However, we don’t want to hard code a new function
call, as our agent might need to go back and forth between itself and
GPT several times until it has found the final answer for the user.

This can be solved in several different ways, e.g. recursion, a
while-loop, or a for-loop. We'll use a good old for-loop for the sake of
simplicity.

## Creating the loop

At the top of the `agent` function, we'll create a loop that lets us run
the entire procedure up to five times.

If we get back `finish_reason: "function_call"` from GPT, we'll just
push the result of the function call to the `messages` array and jump to
the next iteration of the loop, triggering a new request.

If we get `finish_reason: "stop"` back, then GPT has found a suitable
answer, so we'll return the function and cancel the loop.

``` js

for (let i = 0; i < 5; i++) {
  const response = await openai.chat.completions.create({
      model: "gpt-4",
      messages: messages,
      functions: functionDefinitions
  });
  const { finish_reason, message } = response.choices[0];
  
  if (finish_reason === "function_call") {
      const functionName = message.function_call.name;           
      const functionToCall = availableFunctions[functionName];
      const functionArgs = JSON.parse(message.function_call.arguments);
      const functionArgsArr = Object.values(functionArgs);
      const functionResponse = await functionToCall.apply(null, functionArgsArr);
      
      messages.push({
          role: "function",
          name: functionName,
          content: `
          The result of the last function was this: ${JSON.stringify(functionResponse)}
          `
      });
    } else if (finish_reason === "stop") {
    messages.push(message);
    return message.content;   
  }
}
return "The maximum number of iterations has been met without a suitable answer. Please try again with a more specific input.";
```

If we don't see a `finish_reason: "stop"` within our five iterations,
we'll return a message saying we couldn’t find a suitable answer.

## Running the final app

At this point, we are ready to try our app! I'll ask the agent to
suggest some activities based on my location and the current weather.

``` js
const response = await agent("Please suggest some activities based on my location and the current weather.");
console.log(response);
```

Here's what we see in the console (formatted to make it easier to read):

``` js
Based on your current location in Oslo, Norway and the weather (15°C and snowy), 
here are some activity suggestions: 

1. A visit to the Oslo Winter Park for skiing or snowboarding. 
2. Enjoy a cosy day at a local café or restaurant. 
3. Visit one of Oslo's many museums. The Fram Museum or Viking Ship Museum offer interesting insights into Norway’s seafaring history. 
4. Take a stroll in the snowy streets and enjoy the beautiful winter landscape. 
5. Enjoy a nice book by the fireplace in a local library. 
6. Take a fjord sightseeing cruise to enjoy the snowy landscapes.

Always remember to bundle up and stay warm. Enjoy your day!
```

If we peak under the hood, and log out `response.choices[0].message` in
each iteration of the loop, we'll see that GPT has instructed us to use
both our functions before coming up with an answer.

First, it tells us to call the `getLocation` function. Then it tells us
to call the `getCurrentWeather` function with
`"longitude": "10.859", "latitude": "59.955"` passed in as the
arguments. This is data it got back from the first function call we did.

``` js
{role: "assistant", content: null, function_call: {name: "getLocation", arguments: "{}"}}
{role: "assistant", content: null, function_call: {name: "getCurrentWeather", arguments: " { "longitude": "10.859", "latitude": "59.955" }"}}
```

You've now built an AI agent using OpenAI functions and the Node.js SDK! If you're looking for an extra challenge, consider enhancing this app. For example, you could add a function that fetches up-to-date information on events and activities in the user's location.

Happy coding!
