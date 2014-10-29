#Quaderno-Stripe.js

A general purpose library to create Stripe subscriptions, automating the taxes calculation and the Quaderno integration.

##Installing the Quaderno with Stripe js library

This tutorial helps you to integrate Quaderno with Stripe in order to create forms prepared to create subscriptions.

At a high level, here's what you'll accomplish in this tutorial:

1. Set the form and collect credit card information with Stripe.js
2. Convert those details to what Stripe call a single-use toke
4. Create the Stripe subscription via Quaderno, and send the customer id with the rest of your form, to your server

##Step 1: Setting the form basics and collecting credit card information

First, include Stripe.js and quaderno-stripe.js in the page:

    <script type="text/javascript" src="https://js.stripe.com/v2/"></script>
    <script type="text/javascript" src="http://js.quaderno.io/v1/"></script>

To prevent problems with some older browsers, we recommend putting the script tag in the `<head>` tag of your page, or as a direct descendant of the `<body>` at the end of your page.

In addition to your classic Stripe form, you must add some data to the form:

* **key:** (Mandatory) your Stripe publishable key.
* **plan:** (Mandatory) the plan id.
* **taxes:** (Optional) tells how to calculate the taxes. Can be "included" or "excluded". If not present, it will be calculated as "excluded" as default.
* **amount:** (Optional) the amount of the plan in cents. It is only used to inform the customer with live taxes calculations. 

If you add the `amount` data and want to show live previews to the customer, you can add the classes `quaderno-subtotal`, `quaderno-taxes`, and `quaderno-total` to any DOM element to modify its inner HTML. For example:

    $<span class="quaderno-subtotal"></span>
    $<span class="quaderno-taxes"></span>
    $<span class="quaderno-total"></span>

In order to calculate the right tax for your customer and create correct contacts in Quaderno it is also necessary to send a little bit more data than in a regular Stripe form. A complete Quaderno with Stripe form would look like this:

    <form action="" method="POST" id="payment-form" data-key="my_stripe_publishable_key" data-plan="my_plan_id" data-taxes="excluded" data-amount="1599">
        <span class="payment-errors"></span>

        <!-- Basic Stripe form fields -->
        <div class="form-row">
            <label>
              <span>Card Number</span>
              <input type="text" size="20" data-stripe="number"/>
            </label>
        </div>
        <div class="form-row">
            <label>
                <span>CVC</span>
                <input type="text" size="4" data-stripe="cvc"/>
            </label>
        </div>
        <div class="form-row">
            <label>
                <span>Expiration (MM/YYYY)</span>
                <input type="text" size="2" data-stripe="exp-month"/>
            </label>
            <span> / </span>
            <input type="text" size="4" data-stripe="exp-year"/>
        </div>

        <!-- Additional fields for Quaderno -->
        <label>
            * First Name/ Company name:
            <input data-stripe="first_name"/>
        </label>
        <label>
            Last Name:
            <input data-stripe="last_name"/>
        </label>
        <label>
            Email: 
            <input data-stripe="email"/>
        </label>
        <label>
            Street Line 1:
            <input data-stripe="street_line_1"/>
        </label>
        <label>
            Street Line 2:
            <input data-stripe="street_line_2"/>
        </label>
        <label>
            City:
            <input data-stripe="city"/>
        </label>
        <label>
            Region:
            <input data-stripe="region"/>
        </label>
        <label>
            * Country:
            <select data-stripe="country">
            ...
            </select>
        </label>
        <label>
            Postal Code:        
            <input data-stripe="postal_code"/>
        </label>
        <label>
            Tax ID:
            <input data-stripe="tax_id"/>
        </label>
        <button type="submit">Submit Payment</button>
    </form>

Notice that the inputs also has the data-stripe attribute. It is mandatory to include this attribute in at least the **first name** input to prevent unexpected results. By not including it in the **last name**, **country**, **postal code** or **tax id** will result in not exact taxes calculations due to lack of information. 

And that's it. By adding the Quaderno js library it will initialize the Stripe library for you. If you want, you can add a name to those extra field to submit them to your server.

##Step 2: Create a single use token

Next, we will want to create a single-use token that can be used to represent the credit card information your customer enters. Note that you should not store or attempt to reuse single-use tokens. After the code we just added, in a separated script tag, we'll add an event handler to our form. We want to capture the **submit** event, and then use the credit card information to create a single-use token. 

    <script type="text/javascript">
        jQuery(function($) {
            $('#payment-form').submit(function(event) {
                var $form = $(this);

                // Disable the submit button to prevent repeated clicks
                $form.find('button').prop('disabled', true);

                Stripe.card.createToken($form, stripeResponseHandler);

                // Prevent the form from submitting with the default action
                return false;
            });
        });
    </script>


The important code to notice is the call to **Stripe.card.createToken**. The first argument is the form element containing credit card data entered by the user. The relevant values are fetched from their associated inputs using the *data-stripe* attribute specified in the first example.

You should provide at least the card number and expiration info. The complete list of fields you can provide is available in the [Stripe.js documentation](https://stripe.com/docs/stripe.js#createToken).

The second argument **stripeResponseHandler** is a callback that handles the response from Stripe. **createToken** is an asynchronous call – it returns immediately and invokes stripeResponseHandler when it receives a response from Stripe's servers. Whatever function you pass should take two arguments, status and response:

* status is one of the status codes described in the [Stripe API docs](https://stripe.com/docs/api#errors).
* response is an Object with these properties:

---
    {  
        id: "tok_u5dg20Gra", // String of token identifier,  
        card: {...}, // Dictionary of the card used to create the token  
        created: 1414143837, // Integer of date token was created  
        currency: "usd", // String currency that the token was created in  
        livemode: true, // Boolean of whether this token was created with a live or test API key  
        object: "token", // String identifier of the type of object, always "token"  
        used: false // Boolean of whether this token has been used  
    }  


##Step 3: Create the Stripe subscription via Quaderno and send the data to the server

After retrieving the response from ** Stripe.card.createToken** in  **stripeResponseHandler**. In the example, **stripeResponseHandler** works as follows:

* If the card information entered by the user returned an error, it gets displayed on the page.
* If no errors were returned (i.e. a single-use token was created successfully), add the returned token to the form in the stripeToken field and then create the subscription for the customer via Quaderno by calling **Quaderno.createSubscription**.

The code would be:

    function stripeResponseHandler(status, response) {
        var $form = $('#payment-form');
        if (response.error) {
            // Show the errors on the form
            $form.find('.payment-errors').text(response.error.message);
            $form.find('button').prop('disabled', false);
        } else {
            // response contains id and card, which contains additional card details
            var token = response.id;
            // Insert the token into the form so it gets submitted to the server
            $form.append($('<input type="hidden" data-stripe="card_token name="stripeToken" />').val(token));
            Quaderno.createSubscription({
                success: quadernoSuccessHandler(status, response), 
                error: quadernoErrorHandler(status, response)
            });
        }
    };

Please note it is **mandatory** to create the input with the data-stripe="card_token" to make things work properly.

After retrieving the card token, now we are ready to create the subscription via Quaderno. The important call is the **Quaderno.createSubscription**.

**Quaderno.createSubscription** is an ajax function and accepts as argument a Javascript object which should contains three possible handlers. In the example the success handler is **quadernoSuccessHandler** and the error handler **quadernoErrorHandler**.

* success: If Quaderno manages to create the subscription with no errors at all.
* error: If any error prevents from creating the subscription, this handler will be called.
* complete: Wether if the response is a success or an error, this handler will be called. The programmer is responsible to manage the response status

All the handlers accept two arguments, the status and the response.

* status: The code of the request (200, 201, 401, etc.).
* response: Contains a Javascript object with the following structure:

---
    {
        message: 'The message of the response as a string.',
        customer: 'The id of the created customer in Quaderno (only present if the response was a success).'
    }


* The success handler code would be something like this:

---
    function quadernoSuccessHandler(status, response) {
        $form = $('#payment-form');
        $form.append($('<input type="hidden" name="customerId" />').val(response.customer);
        $form.get(0).submit();
    }

* The error handler code would be like this: 

---
    function quadernoErrorHandler(status, response) {
        $form = $('#payment-form');
        $form.find('button').prop('disabled', false);
        $form.find('.payment-errors').text(response.message);
    }


* If the success handler is called, it will create an input with the customer id and will send the form to your server.
* If the error handler is called it will show the error and reactivate the button

Take a look at the [full example](https://gist.github.com/xxswingxx/483f651d2a282a27f1bd) form to see everything put together.
