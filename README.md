# Masked Phone Numbers
### ⏱ 30 min build time

## Why build a number masking application? 

Online service platforms, such as ridesharing, online food delivery and logistics, facilitate the experience between customers and providers by matching both sides of the transaction to ensure everything runs smoothly and the transaction is completed. That is, everyone's happy :)

Sometimes, though, the experience doesn't quite go to plan and it becomes necessary for customers and providers to talk to or message each other directly. The problem then arises that, for many reasons, both parties may not feel comfortable sharing their personal phone number.

A great solution to this issue is using anonymous proxy phone numbers that mask a user's personal phone number while also protecting the platform's personal contact details. The result, a customer doesn't see their provider's phone number but, instead, a number that belongs to the platform and forwards their call to the provider, and vice versa for providers as well.

In this MessageBird Developer Guide, we'll show you how to build a proxy system to mask phone numbers for our ficticious ridesharing platform, BirdCar, implemented in PHP. The sample application includes a data model for customers, drivers, rides and proxy phone numbers and allows setting up new rides from an admin interface for demonstration purposes.

## Using a Number Pool

Before we dive into building the sample application, let's take a moment to understand the concept of a number pool. The idea is to set up a list of numbers by purchasing one or more [virtual mobile numbers](https://www.messagebird.com/numbers) from MessageBird and adding them to a database. Whenever a ride is created, the BirdCar application will automatically search the pool for one that is available and then assign it.

For simplicity and to allow testing with a single number, BirdCar assigns only one number to each ride, not one for each party. If the customer calls or texts this number, they get connected to the driver. And if the driver rings, the call or text is forwarded to the customer. The incoming caller or message sender identification sent from the network is used to determine which party calls and consequently find the other party's number.

Relying on the caller identification has the additional advantage that you do not have to purchase a new phone number for each transaction. Instead, it is possible to assign the same one to multiple transactions as long as different people are involved. The ride can be looked up based on who is calling. It is also possible to recycle numbers even for the same customer or driver, i.e., returning them to the pool, although we have not implemented this behavior in the sample code. In any case, the number should remain active for some time after a transaction has ended, just in case the driver and customer need to communicate afterwards, for example if the customer has forgotten an item in the driver’s car.


## Getting Started

BirdCar's sample application uses PHP with the [Slim](https://www.slimframework.com/) framework. It also uses a relational database to store the data model. To run the sample application, you need to have PHP installed on your machine. If you're using a Mac, PHP is already installed. For Windows, you can [get it from windows.php.net](https://windows.php.net/download/). Linux users, please check your system's default package manager. You also need Composer, which is available from [getcomposer.org](https://getcomposer.org/download/), to install application dependencies like the [MessageBird SDK for PHP](https://github.com/messagebird/php-rest-api).

We've provided the source code in the [MessageBird Developer Guides GitHub repository](https://github.com/messagebirdguides/masked-numbers-guide-php), so you can either clone the sample application with git or download a ZIP file with the code to your computer.

To install the [MessageBird SDK for PHP](https://github.com/messagebird/php-rest-api) and the framework, open a console pointed at the directory into which you've stored the sample application and run the following command:

````bash
composer install
````

## Prerequisites for Receiving Messages and Calls

### Overview

The BirdCar system receives incoming messages and calls and forwards them. From a high-level viewpoint, receiving with MessageBird is relatively simple: an application defines a _webhook URL_, which you assign to a number purchased on the MessageBird Dashboard using a flow. A [webhook](https://en.wikipedia.org/wiki/Webhook) is a URL on your site that doesn't render a page to users but is like an API endpoint that can be triggered by other servers. Every time someone sends a message to that number, MessageBird collects it and forwards it to the webhook URL, where you can process it.

### Exposing your Development Server with ngrok

When working with webhooks, an external service like MessageBird needs to access your application, so the webhook URL must be public. During development, though, you're typically working in a local development environment that is not publicly available. Thankfully this is not a massive roadblock since various tools and services allow you to quickly expose your development environment by providing a tunnel from a public URL to your local machine. One of the most popular tools is [ngrok](https://ngrok.com).

You can [download ngrok here for free](https://ngrok.com/download) as a single-file binary for almost every operating system, or optionally sign up for an account to access additional features.

You can start a tunnel by providing a local port number on which your application runs. We will run our PHP server on port 8080, so you can launch your tunnel with this command:

````bash
ngrok http 8080
````

After you've launched the tunnel, ngrok displays your temporary public URL along with some other information. We'll need that URL in a minute.

![ngrok](ngrok.png)

Another common tool for tunneling your local machine is [localtunnel.me](https://localtunnel.me), which you can have a look at if you're facing problems with ngrok. It works in virtually the same way but requires you to install [NPM](https://www.npmjs.com/) first.

### Getting an Inbound Number

A requirement for receiving messages and voice calls is a dedicated inbound number. Virtual mobile numbers look and work similar like regular mobile numbers, however, instead of being attached to a mobile device via a SIM card, they live in the cloud and can process incoming SMS and voice calls. Here's how to purchase a MessageBird virtual mobile number:

1. Go to the [Numbers](https://dashboard.messagebird.com/en/numbers) section of your MessageBird account and click **Buy a number**.
2. Choose the country in which you and your customers are located and make sure both the _SMS_ and _Voice_ capabilities are selected.
3. Choose one number from the selection and the duration for which you want to pay now. ![Buy a number screenshot](buy-a-number.png)
4. Confirm by clicking **Buy Number**.

Easy! You have set up your first virtual mobile number.

One is enough for testing, but for real usage of the masked number system, you'd need a larger pool of numbers. Follow the same steps listed above to purchase more.

### Connecting the Number to a Webhook for SMS

So you have a number now, but MessageBird has no idea what to do with it. That's why you need to define a _Flow_ next that ties your number to your webhook. We start with the flow for incoming SMS messages:

1. Go to the [Flow Builder](https://dashboard.messagebird.com/en/flow-builder) section of your MessageBird account. Under _Use a template_, you'll see a list of templates. Find the one named "Call HTTP endpoint with SMS" and click **Try this flow**. ![Create Flow, Step 1](create-flow-1.png)
2. The flow contains two steps. On the first step, the trigger "Incoming SMS", tick the box next to all the numbers dedicated to your number pool and **Save**.  ![Create Flow, Step 2](create-flow-2.png)
4. Click on the second step, "Forward to URL". Choose _POST_ as the method, copy the output from the `ngrok` command in the previous step and add `/webhook` to the end of it - this is the name of the route we use to handle incoming messages. Click **Save**. ![Create Flow, Step 3](create-flow-3.png)
5. Click **Publish** to activate your flow.

### Connecting the Number to a Webhook for Voice

You need to set up a second flow for the same number to process incoming calls as well:

1. Go back to the Flow Builder and click **Create new Flow**.
2. Choose *Create Custom Flow*. ![Create Voice Flow, Step 1](create-flow-voice-1.png)
2. Give your flow a name, such as "Number Proxy for Voice", choose _Phone Call_ as the trigger and choose **Next**. ![Create Voice Flow, Step 2](create-flow-voice-2.png)
3. Configure the trigger step by ticking the boxes next to all the numbers dedicated to your number pool and clicking **Save**.
4. Press the small **+** to add a new step to your flow and choose **Fetch call flow from URL**. ![Create Voice Flow, Step 4](create-flow-voice-4.png)
5. Paste the same ngrok base URL into the form, but this time append `/webhook-voice` to it - this is the name of the route we use to handle incoming calls in our sample application. Click **Save**. ![Create Voice Flow, Step 5](create-flow-voice-5.png)
6. Hit **Publish** and your flow becomes active!

## Configuring the MessageBird SDK

The MessageBird SDK and an API key are necessary to send (and forward) messages. The SDK is listed as a dependency in `composer.json`:

````json
{
    "require" : {
        "messagebird/php-rest-api" : "^1.9.4"
        ...
    }
}
````

An application can access the SDK, which is made available through Composer autoloading, by creating an instance of the `MessageBird\Client` class. The constructor takes a single argument, your API key. For frameworks like Slim you can add the SDK to the dependency injection container:

````php
// Load and initialize MesageBird SDK
$container['messagebird'] = function() {
    return new MessageBird\Client(getenv('MESSAGEBIRD_API_KEY'));
};
````

As it's a bad practice to keep credentials in the source code, we load the API key from an environment variable using `getenv()`. To make the key available in the environment variable we need to initialize Dotenv and then add the key to a `.env` file. We've prepared an `env.example` file in the repository, which you should rename to `.env` and add the required information. Here's an example:

````env
MESSAGEBIRD_API_KEY=YOUR-API-KEY
````

To retrieve or generate an API key, go to the [API access (REST) tab](https://dashboard.messagebird.com/en/developers/access) in the _Developers_ section of your MessageBird account.

## Creating our Data Model and Sample Data

Our BirdCar application uses a relational model. It is configured with SQLite, which is natively available in PHP and stores all data in a simple file, so you don't have to set up a database server for testing. You should be able to reuse most of the SQL, though, if you want to replace this with a different database in production.

We have the following four entities:
- _Customers_, who have a name and a phone number.
- _Drivers_, who also have a name and a phone number.
- _Proxy Numbers_, which are the phone numbers in our pool.
- _Rides_, which have a start, destination, and date and time. Every ride references precisely one _Customer_, _Driver_, and _Proxy Number_ through the use of foreign keys.

Every entity has a database table with an auto-incremented numeric ID as its primary key.

Open the file `init.php` in the repository. It contains four CREATE TABLE queries to set up the data model. Below that, you'll find some INSERT INTO queries to add sample customers, drivers, and proxy numbers. Update those queries like this:
- Provide your name and mobile phone number as a customer.
- Provide another working phone number, such as a secondary phone or a friend's number, as a driver.
- Enter the virtual mobile number you purchased on the MessageBird. If you have more than one, copy the query code for each.

After updating the file, save it and run the following command (if you already have ngrok running open a second command prompt for it):

````bash
php init.php
````

Note that this command only works once. If you make changes and want to recreate the database, you must delete the file `ridesharing.sqlite` that the script creates before rerunning it:

````bash
rm ridesharing.sqlite
php init.php
````

## The Admin Interface

The `$app->get('/')` route in `index.php` and the associated HTML page in `views/admin.html.twig` implement a simple homepage that lists the content from the database and provides a form to add a new ride. For creating a ride, an admin can select a customer and driver from a drop-down, enter start, destination and date and time. The form submits this information to `/createride`.

## Creating a Ride

The `$app->post('/createride')` route defined in `index.php` handles the following steps when creating a new ride:

### Getting Customer and Driver Information

The form fields contain only IDs for customer and driver, so we make a query for each to find all the information which we need in subsequent steps:

````php
// Create a new ride
$app->post('/createride', function($request, $response) {
    // Find customer details
    $stmt = $this->db->prepare('SELECT * FROM customers WHERE id = :id');
    $stmt->execute([ 'id' => $request->getParsedBodyParam('customer') ]);
    $customer = $stmt->fetch();

    // Find driver details
    $stmt = $this->db->prepare('SELECT * FROM drivers WHERE id = :id');
    $stmt->execute([ 'id' => $request->getParsedBodyParam('driver') ]);
    $driver = $stmt->fetch();
````

### Finding a Number

We need to get a number from the pool that was never assigned to a ride for the customer or the driver. To check this, we write a SQL query with two subqueries:
- Find all numbers for rides from the selected customer (subquery 1)
- Find all numbers for rides from the selected driver (subquery 2)
- Find all numbers that are in neither of those lists and return one of them (main query)

The SQL query looks like this in our PHP code:

````php
    // Find a number that has not been used by the driver or the customer
    $stmt = $this->db->prepare('SELECT * FROM proxy_numbers '
        . 'WHERE id NOT IN (SELECT number_id FROM rides WHERE customer_id = :customer) '
        . 'AND id NOT IN (SELECT number_id FROM rides WHERE driver_id = :driver)');
    $stmt->execute([
        'customer' => $customer['id'],
        'driver' => $driver['id']
    ]);
    $proxyNumber = $stmt->fetch();
````

It's possible that no proxy number was found. In that case, we alert the admin that the number pool is depleted and they should buy more numbers:

````php
    if ($proxyNumber === false) {
        // No number found!
        return "No number available! Please extend your pool.";
    }
````

### Storing the Ride

If a number was found, i.e., our query returned a row, we insert a new ride into the database using the information from the form:

````php
    // Store ride in database
    $stmt = $this->db->prepare('INSERT INTO rides (start, destination, datetime, customer_id, driver_id, number_id) VALUES (:start, :destination, :datetime, :customer, :driver, :number)');
    $stmt->execute([
        'start' => $request->getParsedBodyParam('start'),
        'destination' => $request->getParsedBodyParam('destination'),
        'datetime' => $request->getParsedBodyParam('datetime'),
        'customer' => $customer['id'],
        'driver' => $driver['id'],
        'number' => $proxyNumber['id']
    ]);
````

### Notifying Customer and Driver

We send a message to both the customer and the driver to confirm the ride. This message should originate from the proxy number, so they can quickly reply to this message or press call to reach the other party. For sending messages, the MessageBird SDK provides the `messages->create()` function. We need to call the function twice because we're sending two different versions of the message:

````php
    // Prepare message object
    $message = new MessageBird\Objects\Message;
    $message->originator = $proxyNumber['number'];
    
    // Notify the customer
    $message->recipients = [ $customer['number'] ];
    $message->body = $driver['name'] . " will pick you up at " . $request->getParsedBodyParam('datetime') . ". Reply to this message or call this number to contact the driver.";
    try {
        $this->messagebird->messages->create($message);
    } catch (Exception $e) {
        error_log(get_class($e).": ".$e->getMessage());
    }

    // Notify the driver
    $message->recipients = [ $driver['number'] ];
    $message->body = $customer['name'] . " will wait for you at " . $request->getParsedBodyParam('datetime') . ". Reply to this message or call this number to contact the customer.";
    try {
        $this->messagebird->messages->create($message);
    } catch (Exception $e) {
        error_log(get_class($e).": ".$e->getMessage());
    }
````

As you can see, there's only one `MessageBird\Objects\Message` object as it is mutable and can be reused for both messages, only changing the attributes as necessary. If there's any error it is logged to the console in a catch-block, but we do not take any action. In production applications, you should definitely check if the messages were sent successfully and implement some more sophisticated error handling.

## Receiving and Forwarding Messages

When a customer or driver replies to the message confirming their ride, the response should go to the other party. As we have instructed MessageBird to post to `/webhook` we need to implement the `$app->post('/webhook')` route.

First, we read the input sent from MessageBird. We're interested in three fields: originator, payload (i.e., the message text) and recipient (the virtual number to which the user sent their message), so that we can find the ride based on this information:

````php
// Handle incoming messages
$app->post('/webhook', function($request, $response) {
    // Read input sent from MessageBird
    $number = $request->getParsedBodyParam('originator');
    $text = $request->getParsedBodyParam('payload');
    $proxy = $request->getParsedBodyParam('recipient');
````

### Looking up Receiver

To find the ride, we use an SQL query which joins all four tables. We're interested in all entries in which the proxy number matches the `recipient` field from the webhook and the `originator` matches _either_ the driver's number _or_ the customer's number:

````php
    // Find potential rides that fit the numbers
    $stmt = $this->db->prepare('SELECT c.number AS customer_number, d.number AS driver_number, p.number AS proxy_number '
        . 'FROM rides r JOIN customers c ON r.customer_id = c.id JOIN drivers d ON r.driver_id = d.id JOIN proxy_numbers p ON p.id = r.number_id '
        . 'WHERE proxy_number = :proxy AND (driver_number = :number OR customer_number = :number)');
    $stmt->execute([
        'number' => $number,
        'proxy' => $proxy
    ]);
    $row = $stmt->fetch();
````

### Forwarding Message

After we've found the ride based on an _or_-condition, we start preparing a `MessageBird\Objects\Message` object with the proxy number as its originator and the received text as the body. Then we need to check again which party was the actual sender and determine the recipient, i.e., the other party that needs to be specified as the _recipients_ attribute, from there:

````php
    if ($row !== false) {
        // Got a match!

        // Prepare message object
        $message = new MessageBird\Objects\Message;
        $message->originator = $proxy;
        $message->body = $text;

        // Need to find out whether customer or driver sent this and forward to the other side
        if ($number == $row['customer_number'])
            $message->recipients = [ $row['driver_number'] ];
        else
        if ($number == $row['driver_number'])
            $message->recipients = [ $row['customer_number'] ];
````

Then, we use the SDK'S `messages->create()` to send the message and create a log message for both success and failure cases:

````php
        // Forward the message through the MessageBird API
        try {
            $this->messagebird->messages->create($message);
            error_log("Forwarded text from " . $number . " to " . $message->recipients[0]);
        } catch (Exception $e) {
            error_log(get_class($e).": ".$e->getMessage());
        }
````

If we haven't found a ride, we also log a message:

````php
    } else {
        // Cannot match numbers
        error_log("Could not find a ride for customer/driver " . $number . " that uses proxy " . $proxy . ".");
    }
````

## Receiving and Forwarding Voice Calls

When a customer or driver calls the proxy number from which they received the confirmation, the system should transfer the call to the other party. As we have instructed MessageBird to fetch instructions from `/webhook-voice` we need to implement the `$app->get('/webhook-voice')` route. Note that unlike the SMS webhook, where we have configured POST, custom call flows are always retrieved with GET.

First, we read the input sent from MessageBird. We're interested in the source and destination of the call so that we can find the ride based on this information:

````php
// Handle incoming calls
$app->get('/webhook-voice', function($request, $response) {
    // Read input sent from MessageBird
    $number = $request->getQueryParam('source');
    $proxy = $request->getQueryParam('destination');
````

As we will return a new call flow encoded in XML format, we set the response header accordingly and start writing the response:

````php
    // Answer will always be XML
    $response = $response->withHeader('Content-Type', 'application/xml')
        ->write('<?xml version="1.0" encoding="UTF-8"?>');
````

### Looking up Receiver

This works exactly as described for the SMS webhooks, hence the SQL query and surrounding PHP code is mostly a verbatim copy. If you are extending the sample to build a production application it could be a good idea to make a function as an abstraction around it to avoid duplicate code.

### Transferring call

To transfer the call, we return a short snippet of XML to MessageBird, and also log the action to the console:

````php
        // Create call flow to instruct transfer
        error_log("Transferring call to " . $destination);
        $response->write('<Transfer destination="' . $destination . '" mask="true" />');
````

The `<Transfer />` element takes two attributes: _destination_ indicates the number to transfer the call to, which we've determined as described above, and _mask_ instructs MessageBird to use the proxy number instead of the original caller ID.

If we don't find a ride, we return a different XML snippet with a `<Say />` element, which is used to read some instructions to the caller:

````php
    } else {
        // Cannot match numbers
        $response->write('<Say language="en-GB" voice="female">Sorry, we cannot identify your transaction. Make sure you call in from the number you registered.</Say>');
    }
````

This element takes two attributes, _language_ and _voice_, that define the configuration for speech synthesis. The text itself goes between the opening and closing XML element.

## Testing the Application

Check again that you have set up at least one number correctly with two flows to forward both incoming messages and incoming phone calls to an ngrok URL and that the tunnel is still running. Remember, whenever you start a fresh tunnel, you'll get a new URL, so you have to update the flows accordingly. If you [register for an ngrok account](https://dashboard.ngrok.com/user/signup) you can also configure a more permanent URL.

To start the application you have to enter another command, but your existing console window is already busy running your tunnel. Therefore you need to open another one. On a Mac you can press _Command_ + _Tab_ to open a second tab that's already pointed to the correct directory. With other operating systems you may have to resort to open another console window manually. Either way, once you've got a command prompt, type the following to start the application:

````bash
php -S 0.0.0.0:8080 index.php
````

Open http://localhost:8080/ in your browser and create a ride between the customer and driver you configured in `init.php`. If everything worked out correctly, two phones should receive a message. Reply to the incoming message on one phone and you'll receive this reply on the other phone, but magically coming from the proxy number. Wow!

If you didn't get the messages or the forwarding doesn't work, check the console output from PHP to see if there's any problem with the API, such as an incorrect API key or a typo in one of the numbers, and try again.

You can then test voice call forwarding as well: call the proxy number from one phone and magically see the other phone ring.

## Nice work!

You've just built your own number masking system with MessageBird! 

You can now use the flow, code snippets and UI examples from this tutorial as an inspiration to build your own application. Don't forget to download the code from the [MessageBird Developer Guides GitHub repository](https://github.com/messagebirdguides/masked-numbers-guide-php).

## Next steps

Want to build something similar but not quite sure how to get started? Please feel free to let us know at support@messagebird.com, we'd love to help!
# masked-numbers-guide-php
