# bot-test-framework-example
This project provides an example of how to build and run regression tests to validate the behavior of a Cisco Spark Bot.

To test your own bot, you do not need to run this entire project, only clone the subproject https://github.com/ciscospark/spark-emulator.   However, this project does contain a sample bot under test as well as some sample test cases that developers may find instructional as they begin to build test cases for their own bots.

In general the testing works as follows:
  - Bot is configured to talk to an instance of the spark emulator, rather than the actual Cisco Spark For Developers API endpoint.
  - The Cisco Spark Emulator is configured to run in "bot test mode".  When in this mode it inspects incoming requests for an X-Bot-Responses header.   For requests that have this header, the emulator intercepts the responses that would normally be sent back to the testing application and waits for the specified number of bot requests to come in.   Each time a correlated bot request is sent to the emulator (correlation generally is done via the roomId), the request information is captured and added to the body of the stored response.   When the number of bot requests specified in the X-Bot-Responses header are made, the consoldiated response is sent back to the testing framework
  - The Testing Framework (this sample uses Postman, but any framework that can send REST requests and inspect the results will do), then inspects the responses to each request, ensuring that the expected bot requests were made

The following sections walk through how to configure the spark-emulator to run in "bot test mode", how to modify a bot in order to make it "testable", and finally how to write and run tests that emulate spark user actions to ensure that the bot responds as expected.

## Configuring the emulator to run in "bot test mode".
This project includes a link to the https://github.com/ciscospark/spark-emulator project.   It is already configured to run in test mode for the bot in this sample, but this was done by doing the following:

1) Edit [spark-emulator/tokens.json](spark-emulator/tokens.json) to include an identity for the bot under test.   Developers should never specify real authorization tokens in tokens.json, but simply strings that the emulator can validate in order to permit clients to make requests against it.   In our case the bot token looks like this:

```
    "ZYXWVUTSRQPONMLKJIHGFEDCBA9876543210": {   // This is the authrorization token for the bot
        "id": "Y2lzY29zcGFyazovL3VzL1BFT1BMRS9hODYwYmFkZC0wNGZkLTQwYWEtYWFjNS05NmYyYWRhZDE3NTA",
        "emails": [
            "bot@sparkbot.io"
        ],
        "displayName": "Bot",
        "nickName": "bot",
        "firstName": "spark",
        "lastName": "bot",
        "avatar": "https://cdn-images-1.medium.com/max/1000/1*wrYQF1qZ3GePyrVn-Sp0UQ.png",
        "orgId": "Y2lzY29zcGFyazovL3VzL09SR0FOSVpBVElPTi8xZWI2NWZkZi05NjQzLTQxN2YtOTk3NC1hZDcyY2FlMGUxMGY",
        "created": "2017-07-18T00:00:00.000Z",
        "type": "bot"
    },
```

2) Edit [spark-emulator/tokens.json](spark-emulator/tokens.json) to include an indentity for the test framework.  In our case this identity looks like this:
```
    "01234567890123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ": {  // this is the auth token for the test client
        "id": "Y2lzY29zcGFyazovL3VzL1BFT1BMRS85MmIzZGQ5YS02NzVkLTRhNDEtOGM0MS0yYWJkZjg5ZjQ0ZjQ",
        "emails": [
            "postman-test@cisco.com"
        ],
        "displayName": "Mr. Postman",
        "nickName": "Posty",
        "firstName": "Post",
        "lastName": "Man",
        "avatar": "https://cdn-images-1.medium.com/max/1600/1*Iel5Q6qAxgBdl_IHUx3scA.jpeg",
        "orgId": "Y2lzY29zcGFyazovL3VzL09SR0FOSVpBVElPTi8xZWI2NWZkZi05NjQzLTQxN2YtOTk3NC1hZDcyY2FlMGUxMGY",
        "created": "2017-07-18T00:00:00.000Z",
        "type": "person"
    }
```

3) Configure the following environment variables before starting the emulator:
    - BOT_UNDER_TEST -- this is set to the email of the bot being tested.   We use bot@sparbot.io as specified in the tokens.json file in step 1 above.   When this environment variable is set the emulator runs in "bot test mode" and will inspect requests and potentially intercept responses that come from the bot in response to test input.
    - DEBUG="emulator:botTest"  -- (Optional).   When set the emulator writes out the details of each request and information about when responses are being intercepted and modified.   This mode is useful when writing test cases as it allows the developer to see the details of the requests that a bot will make in response to given test input.   

In our case we set these variable in package.json as follows:
```
  "scripts": {
    "start-emulator": "BOT_UNDER_TEST=bot@sparkbot.io DEBUG=emulator:botTest node spark-emulator/server.js",
  },
```
## Running the emulator to run in "bot test mode".
Open a terminal window to run the emulator.

1) Make sure all the emulator dependencies are downloaded.  This is necessary only the first time after installing the project locally:

    ```npm run emulator-dependencies```

2) Start the emulator with the environment variables set as described in the previous section:

    ```npm run start-emulator```

## Preparing the bot for testing.

This project includes a fork of the sparkbotstarter project https://github.com/jpjpjp/sparkbotstarter, which demonstrates how to create a simple bot using the node-flint bot framework.   Testing can work with any Cisco Spark bot written in any framework where the developer has the ability to configure the address of the spark instance that the bot interacts with.    We chose the sparkbotstarter project since the spark-emulator and the postman test cases we use in this example all use node.js or javascript.

Preparing a bot for testing requires the following steps.  These steps are already done for you for this sample, but you will need to do these to your own bot in order to run tests against it.   The details of how this is done will differ depending on which bot framework you are using, and how you configure details such as the webhook URL and auth token for your bot.

1) Configure the bot to use a token specified in the tokens.json file of spark-emulator project.   In our sample we see the following line in [sparkbotstarter/config.json](sparkbottester/config.json):

    ```"token": "ZYXWVUTSRQPONMLKJIHGFEDCBA9876543210"```

    This correlates to the bot identity set in tokens.json as described in the previous section on configuring the emulator.

2) Assuming that you will run the emulator on the same machine where your bot under test is running, and that the bot framework  you use configures the webhooks it nees automatically, configure the bot to configure its webhooks on localhost.   The emulator will then send webhooks directly to the bot under test.   In our sample we see the following line in [sparkbotstarter/config.json](sparkbottester/config.json):

    ```"webhookUrl": "http://localhost"```

3) Configure your bot to send spark requests to the emulator instead of to the public Spark For Developers endpoint.   It is possible that your bot framework does not have a mechanism to do this today and you will need to modify your bot code, or even possibly your bot framework to do this.   In our sample we need to do both.  First the bot code this has been modified by adding the following line in [sparkbotstarter/config.json](sparkbottester/config.json):

    ```"sparkApiUrl": "http://localhost:3210/"```

## Running the bot in test mode.
After ensuring that your bot is configured as described above, make sure that the emulator is up and running.  Open a terminal window to start the bot as follows:

1) Make sure all the bot dependencies are downloaded.  This is necessary only the first time after installing the project locally:

    ```npm run bot-dependencies```

2) As described above, we also need to modify the bot framework to work with the emulator.  Since the framework is not part of this sample package **you must perform this step** after downloading the dependencies.  **This is the one step you must perform in order to run the tests with the sample**

     Modify spark.js in the node_modules/node-sparky/lib directory by changing this line:

    ```
     // api url
    this.apiUrl = 'https://api.ciscospark.com/v1/';
    ```
    
    to

    ```
     /// api url
    this.apiUrl = this.options.sparkApiUrl || 'https://api.ciscospark.com/v1/';
    ```


3) Start the emulator with the environment variables set as described in the previous section:

    ```npm run start-emulator```

## Test Cases

Ok.  We've got the emulator and the bot running, now we can dig into what you came here for...test cases!

The easiest way to understand the test cases is to load them into Postman.  If you don't have or don't like Postman you can likely follow along with the test cases in JSON format as well, but you will need to build your own framework for sending the requests and evaluating the responses.

Load the provided test cases and postman environment into your Postman instance by choosing Import from the File Menu and import the two files [test-cases/SparkBotStarterTests.postman_collection.json](test-cases/SparkBotStarterTests.postman_collection.json), and [test-cases/spark_emulator.postman_environment.json](test-cases/spark_emulator.postman_environment.json).  One file will create a collection called SparkBotStarter Tests.   We'll look at that in detail in a moment.   The other creates an environment called spark_emulator.   Let's look at the environment that we have defined:

- API_URL -- the url where the spark emulator is running, ie http://localhost:3210
- SPARK_TOKEN -- a token specified in tokens.json for the spark emulator
- bot_email -- the email ID of the bot that will be tested.   Specified in [spark-emulator/tokens.json](spark-emulator/tokens.json)
- bot_id -- the id of the bot being tested.  Specified in [spark-emulator/tokens.json](spark-emulator/tokens.json)
- tester_name -- the name of the user running the tests.  This is for a user other than the bot who is specified in [spark-emulator/tokens.json](spark-emulator/tokens.json)
- tester_email -- the email of the user running the tests.  Specified in [spark-emulator/tokens.json](spark-emulator/tokens.json)

The values configured should work out-of-the-box for this sample.

After ensuring that you have activated this environment (and that your bot and spark emulator are running), you are ready to run your first test.   It's easier to follow along with this section if you have postman up and running and are looking at the requests and associated test cases.  If you aren't familiar with Postman, requests are stored in a "Collection".  Each request has a title, a spot to define the request type and endpoint, and under that several tabs.  In our collection we often update the Headers, Body, and the Tests tab.  Tests are written in javascript and allow you to programmatically evaluate the responses to each request.   This blogpost provides a good overview of how Postman based testing works: http://blog.getpostman.com/2017/10/25/writing-tests-in-postman/

You'll find the following in your new SparkBotStarter Tests collection:

1) **Create a group room (and define global test cases)**:   This is a fairly "standard" test case that just creates a space that our test harness and bot will be messaging back and forth in.   The bot is not a member yet, so we don't expect any response from the bot, however there are some interesting global functions defined in the Test tab of the test.   For example:

    ```
    postman.setGlobalVariable("bodyIncludesBotResponsesTest", () => {
    // The response body includes both the response plus the expected bot requests
    var data = JSON.parse(responseBody);
    var expectedResponses = request.headers['x-bot-responses'];
    tests["X-Bot-Responses request header was set to:" + expectedResponses] = expectedResponses !== undefined;
    tests["Response body has a TestFrameworkResponse object"] = data.testFrameworkResponse !== undefined;
    tests["Response body has the botResponses array"] = data.botResponses !== undefined;
    });
    ```
    Here we create a generic bodyIncludesBotResponsesTest which will validate that the response from the emulator includes the body we expect when a response includes a consolidated response body plus subsequent bot requests.

    Our test case also inspects the return value and finds the roomId of the newly created space.   This value is saved in the environment variable "_roomId" which is used by the subsequent tests.

    Its important that this test be run before the others.

2) **Add bot to group room**:   This is also a fairly "standard" test as it just adds the bot to our space.  The SparkStarterBot in this sample project does not respond in any way when being added to a space so there are no expected Bot responses here either.

3) **List memberships (2)**:  This test is also "standard" in that it validates that the Spark emulator returns two memberships, one for the tester account and one for the bot, in the newly created space.

4) **Send /hello message to space**:  Ok, now we are doing something interesting.   In the body of this message we are mentioning the bot since bots can only see messages when they are mentioned in a group space.   The spark emulator does not do a great job of handling emails yet, so we mention the bot explicitly with its id.  Since we want to say "/hello", the body of the message request looks like this:
    ```
    {
    "roomId" : "{{_room}}",
    "text" : "<@personId:{{bot_id}}|Bot> /hello"
    }
    ```
    Note that in Postman's syntax an environment variable enclosed in double brackets is replaced by that variable's value.

    But of course the interesting part of this test is that we've introduced a new header (see the Headers tab) called "X-Bot-Responses" and set the value to 1.   This essentially tells the spark emulator to intercept the response to our messages request and to hold it until it gets one request from the bot.   Since this is happening these tests may take a few seconds to run.   Assuming all is well and the bot is behaving properly the test framework will eventually get a response that looks something like this:

    ```
    {
        "testFrameworkResponse": {
            "id": "Y2lzY29zcGFyazovL2VtL01FU1NBR0UvZTM4MTM4NzgtOTdjMy00YmFmLThhMTItMjUzOWY5MDgxMzU5",
            "roomId": "Y2lzY29zcGFyazovL2VtL1JPT00vOGE4MTZiNjAtMjI3NS00ZjRkLThjNzYtMjEwNWJiMzRhMzkw",
            "roomType": "group",
            "text": "<@personId:Y2lzY29zcGFyazovL3VzL1BFT1BMRS9hODYwYmFkZC0wNGZkLTQwYWEtYWFjNS05NmYyYWRhZDE3NTA|Bot> /hello",
            "personId": "Y2lzY29zcGFyazovL3VzL1BFT1BMRS85MmIzZGQ5YS02NzVkLTRhNDEtOGM0MS0yYWJkZjg5ZjQ0ZjQ",
            "personEmail": "postman-test@cisco.com",
            "created": "2018-03-19T18:01:28.342Z"
        },
        "botResponses": [
            {
                "method": "POST",
                "route": "/messages",
                "body": {
                    "text": "Mr. Postman, you said hello to me!",
                    "roomId": "Y2lzY29zcGFyazovL2VtL1JPT00vOGE4MTZiNjAtMjI3NS00ZjRkLThjNzYtMjEwNWJiMzRhMzkw"
                }
            }
        ]
    }
    ```
    Moving over to the Test tab in postman we can see calls to the functions we defined in step one as we run a total of nine tests on this response.  Let's spend a little time reviewing what is happening with these tests:
    ```
    // First, run the common tests
    eval(globals.commonTests)("Send help message to the bot", 200);

    // Check that we got the combined spark API and bot response payload
    eval(globals.bodyIncludesBotResponsesTest)()

    // Check if we got the expected message from the bot

    var jsonData = JSON.parse(responseBody);
    var testFrameworkResponse = jsonData.testFrameworkResponse;
    var botResponses = jsonData.botResponses;

    var roomId = postman.getEnvironmentVariable("_room");
    var expectedResponses = request.headers['x-bot-responses'];
    if (botResponses.length == expectedResponses) {
        tests["Bot responded with " + expectedResponses + " requests:"] = true;
        var botResponse = jsonData.botResponses[0]
        var testerName = postman.getEnvironmentVariable("tester_name");
        var resultMsg = testerName + ', you said hello to me!';
        eval(globals.botResponseMessageText)(botResponse, roomId, resultMsg);
    } else {
        tests["Bot responded with " + expectedResponses + " requests:"] = false;
    }
    ```
    The first eval of globals.commonTests simply makes sure that we got the expected response code to our original request.  The next eval of the globals.bodyIncludesBotResponsesTest makes sure that the response includes both the testFrameworkResponse object and an array of botResponses, which are actually request(s) that the bot made in response to our test input.

    Finally, the eval of globals.botResponseMessageText, checks to see that the bot request was to the messages endpoint, for our room, and matched our expected text.   This is the meat of the test framework.   Once you understand this you can develop tests for any given input to your bot building up a regression test that you can run whenever your bot changes.

5) **Send /whoami message to the space**:  This test follows the same model set up in the previous test.  For this particular input we expect the bot to reply with two messages so we set X-Bot-Responses to 2.   The test cases look similar to the previous test but since we expect markdown we eval the globals.botResponseMessageMarkdown function two times, once for each response in the botResponses array.

6) **Send /echo message to space**:  Very similar to test 4 with different input and expected output.

7) **Send /leave message to space**:  In this command we expect two bot responses but only one message.  The second response is a DELETE request sent to the memberships API.   The test case validates that this is what was recieved

8) **List memberships (1)**:  This "standard" test doesn't look for any bot responses, it simply validates that the bot is no longer a member of the space.

9) **Delete group room**:  Cleans up our environment.

Thats it.  Spend some time running the tests, looking at the responses and the output of the tests.   Just as Postman has a Tests tab in the Request section, it has one in the Response section as well.  I find it handy to first look at the response, and then switch over to the Tests tab to see which tests passed and which tests failed.

## Building your own tests
Once you have the hang of how Postman tests work, and what the X-Bot-Responses header does, you should be able to start writing test cases for your own bot.   You'll need to configure your own bot to talk to the spark emulator as described above.   So far we have only configured bots using the node-flint framework, but we'd love to get feedback or samples from users on how to do this with other frameworks.

Once you have your bot talking to the spark emulator, I find it useful to run the emulator with DEBUG=emulator:botTest (which happens automatically if you start the emulator using the npm run start-emulator command).  In this mode you can see all of the requests being sent to it and watch the responses the the bot sends after input from Postman.   Using this you can start to experiment with setting the X-Bot-Responses header and creating test cases that will validate the response specific to your bot.

## Running your regression tests
Once you are confident that your tests are working properly, its time to get in the habit of running the tests regularly.   Postman has a tool called Runner (click the Runner button on the main widow) which allows you to select a collection and environment and run all the tests automatically.  Postman also supplies a CLI tool called newman which allows you to run tests from the command line.   This sample project includes the following npm command to run the tests in CLI mode:

    ```npm test```

Just make sure that the emulator and configured bot are both running before running the test.

## Limitations
There are still several known limitations to testing bots with this framework.   Among them are:
- Correlation of bot requests to test input is currently done only via roomId (or bot membership in room for membership events).   This means that the test framework cannot send a second request that impacts the same room as the first request before the first response is returned.
- There is currently no way correlate bot behaviors that dont take place in the same room as the test request.   This means that if (for example) a bot responds to a message being sent in one room by sending messages to another room, there is no way to capture this in the framwork today.
- There is currently no way to test external events that generate bot responses since the test framework depends on the initial request coming into the spark emulator

We hope to extend the spark emulator and work with bot framework developers to create a more precise way to correlate test input with subsequent bot requests.   Watch this space!

If you find other limitations please report them as issues. 
