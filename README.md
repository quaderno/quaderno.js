#Quaderno.js

A library to create Stripe subscriptions and charges, calculate taxes on the fly, and send beautiful invoices. This is the documentation for the v2, if you want the v1 you can find it [here](v1/README.md)

##Installing the Quaderno.js library

This tutorial helps you to integrate quaderno.js in your app in order to create subscriptions with taxes (if needed).

Here's what you'll accomplish in this tutorial:

1. Set up the form and collect credit card information with Stripe.js
2. Convert those details to what Stripe call a single-use token
3. Create the Stripe subscription via Quaderno and send the data to your server

###Step 1: Setting up the form and collecting credit card information

First, include stripe.js and quaderno.js in the page:

```
<script src="https://js.stripe.com/v2/"></script>
<script src="https://js.quaderno.io/v2/"></script>
```

To prevent problems with some older browsers, we recommend putting the script tag in the `<head>` tag of your page, or as a direct descendant of the `<body>` at the end of your page.

You must add some extra data to your classic Stripe form:

* **key:** (mandatory) Your Quaderno publishable key. You can be find it by logging into Quaderno and clicking **Settings > API**
* **plan:** (mandatory) the plan id.
* **taxes:** (optional) tells how to calculate the taxes. Can be "included" or "excluded". If not present, it will be calculated as "excluded" as default.
* **amount:** (optional) the amount of the plan in cents. This is only used to show customers live tax calculations.

```html
<form action="" method="POST" id="payment-form"
  data-key="YOUR_PUBLISHABLE_KEY_FOR_QUADERNO"
  data-plan="YOUR_PLAN_ID"
  data-taxes="excluded"
  data-amount="900">
  ...
</form>
```

If you add the `amount` data and want to show live tax previews to the customer, you can add the classes `quaderno-subtotal`, `quaderno-taxes`, and `quaderno-total` to any DOM element to modify its inner HTML. For example:

```html
$<span class="quaderno-subtotal"></span>
$<span class="quaderno-taxes"></span>
$<span class="quaderno-total"></span>
```

In order to calculate the right tax for your customer and create correct contacts in Quaderno it's necessary to add some extra  extra inputs. The extra inputs have a `data-quaderno` attribute. It is mandatory to include the data-quaderno attribute in at least the **first name** input to prevent unexpected results. Including the data-quaderno attribute in the **last name**, **country**, **postal code** or **VAT number** is necessary for exact tax calculation.
Also, if necessary, you can specify the per-user pricing by setting an optional input with the data-quaderno **quantity**. By default it is 1.

A complete Quaderno with Stripe form would look like the example below:

```html
<form action="" method="POST" id="payment-form" data-key="YOUR_QUADERNO_PUBLISHABLE_KEY" data-plan="YOUR_PLAN_ID" data-taxes="excluded" data-amount="900">
    <span class="payment-errors"></span>

    <input type="hidden" data-quaderno="quantity" value=1 />

    <!-- Billing form fields -->
    <fieldset>
      <legend>Billing Data</legend>
      <div class="form-row">
        <label>
          <span>* First Name / Company Name</span>
          <input data-quaderno="first-name"/>
        </label>
      </div>

      <div class="form-row">
        <label>
          <span>Last Name</span>
          <input data-quaderno="last-name"/>
        </label>
      </div>

      <div class="form-row">
        <label>
          <span>Email</span>
          <input data-quaderno="email"/>
        </label>
      </div>

      <div class="form-row">
        <label>
          <span>Street Line 1</span>
          <input data-quaderno="street-line-1"/>
        </label>
      </div>

      <div class="form-row">
        <label>
          <span>Street Line 2</span>
          <input data-quaderno="street-line-2"/>
        </label>
      </div>

      <div class="form-row">
        <label>
          <span>City</span>
          <input data-quaderno="city"/>
        </label>
      </div>

      <div class="form-row">
        <label>
          <span>Postal Code</span>
          <input data-quaderno="postal-code"/>
        </label>
      </div>

      <div class="form-row">
        <label>
          <span>Region</span>
          <input data-quaderno="region"/>
        </label>
      </div>

      <div class="form-row">
        <label>
          <span>Country</span>
          <select data-quaderno="country">
          ...
          </select>
        </label>
      </div>

      <div class="form-row">
        <label>
          <span>VAT Number</span>
          <input data-quaderno="vat-number"/>
        </label>
      </div>

      <div class="form-row">
        <label>
          <span>Coupon</span>
          <input data-quaderno="coupon"/>
        </label>
      </div>
    </fieldset>

    <!-- Stripe form fields -->
    <fieldset>
      <legend>Credit Card Data</legend>
      <div class="form-row">
        <label>
          <span>* Card Number</span>
          <input data-stripe="number"/>
        </label>
      </div>

      <div class="form-row">
        <label>
          <span>* CVC</span>
          <input data-stripe="cvc" type="password"/>
      </label>
      </div>

      <div class="form-row">
        <label>
          <span>* Expiration (MM/YYYY)</span>
          <input type="text" size="2" data-stripe="exp-month"/>
        </label>
        <span> / </span>
        <input type="text" size="4" data-stripe="exp-year"/>
      </div>
    </fieldset>

    <button type="submit">Submit Payment</button>
</form>
```

And that's it! If you want, you can add a name to those extra field to submit them to your server.

###Step 2: Create a single use token

Next, we will want to create a single-use token that can be used to represent the credit card information your customer enters. Note that you should not store or attempt to reuse single-use tokens. After the code we just added, in a separate script tag, we'll add an event handler to our form. We want to capture the **submit** event, and then use the credit card information to create a single-use token.

```html
<script>
  jQuery(function($) {
    $('#payment-form').submit(function(event) {
      var $form = $(this);
      // Ste the Stripe publishable key
      Stripe.setPublishableKey($form.data('gateway-key'))

      // Disable the submit button to prevent repeated clicks
      $form.find('button').prop('disabled', true);
      Stripe.card.createToken($form, stripeResponseHandler);

      // Prevent the form from submitting with the default action
      return false;
    });
  });
</script>
```

The important code to notice is the call to **Stripe.card.createToken**. The first argument is the form element containing credit card data entered by the user. The relevant values are fetched from their associated inputs using the *data-stripe* attribute specified in the first example.

You should provide at least the card number and expiration info. The complete list of fields you can provide is available in the [Stripe.js documentation](https://stripe.com/docs/stripe.js#createToken).

The second argument **stripeResponseHandler** is a callback that handles the response from Stripe. **createToken** is an asynchronous call – it returns immediately and invokes stripeResponseHandler when it receives a response from Stripe's servers. Whatever function you pass should take two arguments, status and response:

* status is one of the status codes described in the [Stripe API docs](https://stripe.com/docs/api#errors).
* response is an Object with these properties:

```js
{
  id: "tok_u5dg20Gra", // String, token identifier,
  card: {...}, // Object, the card used to create the token
  created: 1414143837, // Number, date token was created
  currency: "usd", // String, currency that the token was created in
  livemode: true, // Boolean, whether this token was created with a live or test API key
  object: "token", // String, identifier of the type of object, always "token"
  used: false // Boolean, whether this token has been used
}
```

###Step 3: Create the Stripe subscription via Quaderno and send the data to the server

After retrieving the response from ** Stripe.card.createToken** in  **stripeResponseHandler**. In the example, **stripeResponseHandler** works as follows:

* If the card information entered by the user returned an error, it gets displayed on the page.
* If no errors were returned (i.e. a single-use token was created successfully), add the returned token to the form in the stripeToken field and then create the subscription for the customer via Quaderno by calling **Quaderno.createSubscription**.

The code would be:

```js
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
    $form.append($('<input type="hidden" data-quaderno="stripeToken" />').val(token));
    Quaderno.createSubscription({
      success: quadernoSuccessHandler(status, response),
      error: quadernoErrorHandler(status, response)
    });
  }
};
```

Please note it is **mandatory** to create the input with the data-quaderno="stripeToken" to make things work properly.

After retrieving the card token, now we are ready to create the subscription via Quaderno. The important call is the **Quaderno.createSubscription**.

**Quaderno.createSubscription** is an ajax function and accepts as argument a Javascript object which should contains three possible handlers. In the example the success handler is **quadernoSuccessHandler** and the error handler **quadernoErrorHandler**.

* success: If Quaderno manages to create the subscription with no errors at all.
* error: If any error prevents from creating the subscription, this handler will be called.
* complete: Whether the response is a success or an error, this handler will be called. The programmer is responsible to manage the response status.

All the handlers accept two arguments, the status and the response.

* status: The code of the request (200, 201, 401, etc.).
* response: Contains a Javascript object with the following structure:

```js
{
  message: 'The message of the response as a string.',
  details: 'A JWT encoded with your Quaderno private key containing the transaction details'
}
```

The details attribute once decoded using your **Quaderno private key** is a hash with the following attributes:

```ruby
{
  gateway: 'The payment gateway used for the transaction.',
  type: 'The type of the transaction (subscription or charge)',
  customer: 'The id of the customer generated for the transaction',
  transaction:'The transaction id (the subscription id or the charge id)',
  iat: 'A UNIX timestamp'

}

```

* The success handler code would be something like this:

```js
function quadernoSuccessHandler(status, response) {
  $form = $('#payment-form');
  $form.append($('<input type="hidden" name="transactionDetails" data-quaderno="transactionDetails" />').val(response.details);
  $form.get(0).submit();
}
```

* The error handler code would be like this:

```js
function quadernoErrorHandler(status, response) {
  $form = $('#payment-form');
  $form.find('button').prop('disabled', false);
  $form.find('.payment-errors').text(response.message);
}
```

* If the success handler is called, it will create an input with the transaction details and will send the form to your server.
* If the error handler is called it will show the error and reactivate the button

Take a look at the [full example](subscriptions_example.html) form to see everything put together.


### Creating single charges

Creating single charges is very similar to creating subscriptions, but in order to prevent fraud some calculations must be made in your backend prior rendering the payment form.

Before showing the payment form to your customer, you must encode a [JSON Web Token](http://jwt.io/) (JWT) with your **Quaderno private key** (**not** your Quaderno public key). You can find the private key under **Settings > API** in Quaderno. This JWT should contain, as minimum data, the amount of the charge in cents and an issuedAt timestamp (called `iat` for short) which defines the seconds since the UNIX epoch. For example:

```json
{
  "amount":1000,
  "iat":1421753188
}
```

* **amount:** (mandatory) amount of the transaction. Quaderno will handle the taxes when creating the charge.
* **iat** (mandatory) necessary to prevent the reuse of the generated JWT. Quaderno will give a 10 minutes window for the generated JWT, so that's pretty much the time that your customer has to fill the form and make the payment.
* **currency** (optional) if not set, the charge will be made using 'USD'.
* **po_number** (optional) a unique identifier of the order. You can use it to check if your order data meets the invoice generated in Quaderno.

Then codify it as a JWT with your Stripe access token. An example with ruby and the  `jwt` gem (https://github.com/progrium/ruby-jwt):

```ruby
jwt = JWT.encode({"amount" => 1000, "iat" => 1421753188}, 'YOUR_SECRET_KEY_FOR_QUADERNO')

puts jwt #=> "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.IntcImFtb3VudFwiOjEwMDAsIFwiaWF0XCI6MTQyMTc1MzE4OH0i.da1E9xAQDoX6cDhNMkJuRkJPpeAOUMTBACsD--pr4w4" 
```

If you are working with **node.js**, **[@mikemaccana](https://github.com/mikemaccana)** has made a nice plugin to get the necessary JWT in a much easier way. You can read more about it [here](https://www.npmjs.com/package/quaderno-server).

Once you have generated the JWT, you can render the payment form, which should be very similar to the one used for the subscriptions example, but instead adding the `data-plan`, now you should add the `data-charge` with the JWT as the value.

```html
<form action="" method="POST" id="payment-form"
  data-key="YOUR_PUBLISHABLE_KEY_FOR_QUADERNO"
  data-charge="eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.IntcImFtb3VudFwiOjEwMDAsIFwiaWF0XCI6MTQyMTc1MzE4OH0i.da1E9xAQDoX6cDhNMkJuRkJPpeAOUMTBACsD--pr4w4"
  data-taxes="excluded"
  data-amount="900">
  ...
</form>
```

And instead calling the `Quaderno.createSubscription` method you should call `Quaderno.createCharge`, which accepts the same arguments as the `createSubscription`.

```js
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
    $form.append($('<input type="hidden" data-stripe="stripeToken" />').val(token));
    Quaderno.createCharge({
      success: quadernoSuccessHandler(status, response),
      error: quadernoErrorHandler(status, response)
    });
  }
};
```


You can see a full example of single charges creation [here](charges_example.html).

## Advanced usage

### Programatically init Quaderno

By default, if you include the identifier `payment-form` like in the [example](example.html), Quaderno automatically will associate the necessary callbacks to your form. Nevertheless if the form is not present in the DOM when the page is loaded or if you want to use a different selector you can do it by calling:

```javascript
Quaderno.init('#custom-form-selector')
```

### Calculate taxes on demand

By default, in order to refresh the taxes calculations when the form is modified, Quaderno will associate the event `change` handler on the inputs with the following values for `data-quaderno` (if present) :

* `company-name`
* `country`
* `postal-code`
* `vat-number`
* `quantity`

If you want to force a taxes calculation or associate your custom events to a taxes calculation, you can do it like this:

```javascript
  Quaderno.calculateTaxes(callbacks)
```

Because this is an asynchronous ajax function, if you want to execute something only when the taxes are calculated, we provide the `callbacks` argument, which is an optional JS object which could contain three types of callbacks:

* success: Executed if the taxes calculation was a success.
* error: Executed if the taxes calculations failed (not valid stripe publishable key, for example) .
* complete: Executed no matter if the response was a success or an error.

All the above callbacks accept two arguments, the status code and the response. For example:

```javascript
  Quaderno.calculateTaxes({ success: function(statusCode, response) { alert(statusCode); } }); // => Show a 200 in the alert
```

### Listen the "taxCalculated.Quaderno" event

Whenever a tax is succesfully calculated, the event `taxCalculated.Quaderno` is dispatched on the payment form. It's useful if you want to bind some custom actions to the automatic taxes calculations made by Quaderno.js.

```javascript
  // With plain JavaScript
  document.getElementById("payment-form").addEventListener('taxCalculated.Quaderno', function(data){
    alert(data.detail.message) // "A tax has been calculated"
  })

  // With JQuery
  $('#payment-form').on('taxCalculated.Quaderno', function(data){
    alert(data.namespace + ": " + data.detail.message) // "Quaderno: A tax has been calculated"
  })
```

The only argument of the handler, `data`, is a JS object which contains a child object called `detail` which in turn has another two childs:

* **tax**: the calculated tax (exactly the same value returned by `Quaderno.readQuadernoTaxes()`)
* **message**: just a string with the event associated message ("A tax has been calculated").


### Read last calculated tax

If you just want to check the values of the last calculated tax, you can always call do:

```javascript
Quaderno.readQuadernoTaxes(); // => Object {name: "IVA", rate: 22, notes: null}
Quaderno.readQuadernoTaxes().name; // => "IVA"
Quaderno.readQuadernoTaxes().rate; // => 22
…
```

## License

The MIT License (MIT)

Copyright (c) 2014-2015 Quaderno

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
