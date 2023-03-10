# Setup for M-pesa stk push and query on Rails APIs.

Rails App API to demo the basics of a simple stk push and stk push query (checking if payment is successfull or might have encounted any errors) on M-pesa.

### Setup

- In order to implement it, install the following:

#### Daraja Safaricom Developers Portal

- Get started by login in or create [Daraja Developer](https://developer.safaricom.co.ke/) free Account.
- On apps tab, create a new sandbox app and give it your preferd name. Proceed to tick all the check boxes and click on create app.
- You will be redirected to the app details page where you will find your consumer key and consumer secret. This will be crucial later during setup.
- Navigate to the APIs tab and on M-pesa Express click on Simulate, on the input prompt select the app you just created.
- Scroll down and click on test credentials. The initiator password and passkey will crucial also later.

#### Ngrok

- Login or create [ngrok](https://ngrok.com/) free account.
- Install on ubuntu by `sudo snap install ngrok` or download from website.
- Connect your account to ngrok run `ngrok authtoken <your authtoken>`.

#### Rails

- Create a new rails app `rails new <name> --api --minimal`.
- Install or Add Gems below.

```ruby
gem 'rest-client'
gem 'rack-cors'
```

- Run `bundle install`.

- Create M-pesa resource. Run `rails g resource Mpesa phoneNumber amount checkoutRequestID  merchantRequestID mpesaReceiptNumber --no-test-framework`.
- Also add Access Token. Run ` rails g model AccessToken token --no-test-framework`.
- Run `rails db:migrate`

### Configurations

- Navigate to `config/environments/development.rb` and add the following code.

```ruby
config.hosts << /[a-z0-9]+\.ngrok\.io/
```

> **This will allow us to access our rails app from ngrok**.

- Navigate to `config/initializers/cors.rb` and add the following code or uncomment the existing code.

```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "*"

    resource "*",
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end

end
```

> **Be sure to replace origins 'example.com' with origins '\*' if you uncomment existing code.**

### Config Environment Variables

- Inside the config folder create a file `config/local_env.yml` and add the following code.

```ruby
MPESA_CONSUMER_KEY: "<your consumer key>"
MPESA_CONSUMER_SECRET: "<your consumer secret>"
MPESA_SHORTCODE: "<your mpesa shortcode>"
MPESA_INITIATOR_NAME: "testapi"
MPESA_PASSKEY: "<your passkey>"
MPESA_INITIATOR_PASSWORD: "<your initiator password>"
CALLBACK_URL: "<your ngrok url>"
REGISTER_URL: "https://sandbox.safaricom.co.ke/mpesa/c2b/v1/registerurl"
```

> **IMPORTANT:** Add the local_env.yml to the .gitignore file to hide your Daraja Consumer Key and Secret.

### Note about the CALLBACK_URL.

- To aquire callback url, run your rails server `rails s` and copy the url `http://127.0.0.1:3000` from the terminal.
- Open a new terminal for ngrok and run `ngrok http <port number>` and replace `<port number>` with the port number from your rails server `http://127.0.0.1:3000` hence generating a url that you can use as your callback_url.
- My example: `ngrok http http://127.0.0.1:3000` and the url generated is `https://d0e6-154-154-16-76.in.ngrok.io`

  > **Note** that the url generated by ngrok changes every time you run it, so you will need to update your local_env.yml file with the new url every time you run ngrok.

- Navigate to ngrok url, to open the link, click on visit site which should take you to your rails app. If you get a `Blocked Host Error`, check out this stackoverflow [solutions](https://stackoverflow.com/questions/53878453/upgraded-rails-to-6-getting-blocked-host-error).
- The solution that worked for my test is to replace `config.hosts << /[a-z0-9]+\.ngrok\.io/` with ` config.hosts.clear` in `config/environments/development.rb`, though this is not recommended for production.

- To load rails on our environment variables, add the following code to `config/application.rb`.

```ruby
config.before_configuration do
  env_file = File.join(Rails.root, 'config', 'local_env.yml')
  YAML.load(File.open(env_file)).each do |key, value|
    ENV[key.to_s] = value
  end if File.exists?(env_file)
end
```

### Implementing The Main Code.

- To get started, we start with the private methods to generate and get an access token from the Authorization API.
- Generate Access Token Request ----> Gives you a time bound access token to call allowed APIs it provides you with an access token.

- Get Access Token ----> Used to check if generate_acces_token_request is successful or not then it reads the responses and extracts the access token from the response and saves it to the database.

- Add the this code to `app/controllers/mpesas_controller.rb`.

- First add rest-client gem.

```ruby
require 'rest-client'
```

> generate_access_token_request

```ruby
private

    def generate_access_token_request
        @url = "https://sandbox.safaricom.co.ke/oauth/v1/generate?grant_type=client_credentials"
        @consumer_key = ENV['MPESA_CONSUMER_KEY']
        @consumer_secret = ENV['MPESA_CONSUMER_SECRET']
        @userpass = Base64::strict_encode64("#{@consumer_key}:#{@consumer_secret}")
        headers = {
            Authorization: "Bearer #{@userpass}"
        }
        res = RestClient::Request.execute( url: @url, method: :get, headers: {
            Authorization: "Basic #{@userpass}"
        })
        res
    end
```

> get_access_token

```ruby
private

...

    def get_access_token

        res = generate_access_token_request()
        if res.code != 200
            r = generate_access_token_request()
            if res.code != 200
                raise MpesaError('Unable to generate access token')
            end
        end

        body = JSON.parse(res, { symbolize_names: true })
        token = body[:access_token]
        AccessToken.destroy_all()
        AccessToken.create!(token: token)
        token
    end
```

#### Stk Push Request

- On APIs ---> M-pesa Express you can [simulate](https://developer.safaricom.co.ke/APIs/MpesaExpressSimulate) a stk push request by selecting your app and changing Party A and Phone Number to your phone number.

```
  payload = {
    BusinessShortCode --> The organization shortcode used to receive the transaction.
    Password --> Should be encoded with base64 format (business_short_code + mpesa_passkey+timestamp)
    Timestamp --> The timestamp of the transaction in the format ???%Y%m%d%H%M%S???
    TransactionType --> The type of transaction (CustomerPayBillOnline or CustomerBuyGoodsOnline)
    Amount --> The amount being transacted
    PartyA --> The phone number sending the money.
    PartyB --> The organization shortcode receiving the funds.Can be the same as the business shortcode.
    PhoneNumber --> The mobile number to receive the STK push.Can be the same as Party A.
    CallBackURL --> The url to where responses from M-Pesa will be sent to. Should be valid and secure.
    AccountReference --> Value displayed to the customer in the STK Pin prompt message.
    TransactionDesc --> A description of the transaction.
  }
```

- Read more on the [documentation](https://developer.safaricom.co.ke/Documentation) ---> Lipa Na M-pesa Online API ---> Request Parameter Definition.
- Add the following code to `app/controllers/mpesa_controller.rb`.

```ruby
    def stkpush
        phoneNumber = params[:phoneNumber]
        amount = params[:amount]

        url = "https://sandbox.safaricom.co.ke/mpesa/stkpush/v1/processrequest"
        timestamp = "#{Time.now.strftime "%Y%m%d%H%M%S"}"
        business_short_code = ENV["MPESA_SHORTCODE"]
        password = Base64.strict_encode64("#{business_short_code}#{ENV["MPESA_PASSKEY"]}#{timestamp}")
        payload = {
            'BusinessShortCode': business_short_code,
            'Password': password,
            'Timestamp': timestamp,
            'TransactionType': "CustomerPayBillOnline",
            'Amount': amount,
            'PartyA': phoneNumber,
            'PartyB': business_short_code,
            'PhoneNumber': phoneNumber,
            'CallBackURL': "#{ENV["CALLBACK_URL"]}/callback_url",
            'AccountReference': 'Mpesa_ROR',
            'TransactionDesc': "Payment for Mpesa_ROR"
        }.to_json

        headers = {
            Content_type: 'application/json',
            Authorization: "Bearer #{get_access_token}"
        }

        response = RestClient::Request.new({
            method: :post,
            url: url,
            payload: payload,
            headers: headers
            }).execute do |response, request|
                case response.code
                when 500
                    [ :error, JSON.parse(response.to_str) ]
                when 400
                    [ :error, JSON.parse(response.to_str) ]
                when 200
                    [ :success, JSON.parse(response.to_str) ]
                else
                    fail "Invalid response #{response.to_str} received."
                end
            end
        render json: response


    end
```

- Navigate to `config/routes.rb` and add this code.

```ruby
post 'stkpush', to: 'mpesas#stkpush'
```

- Open up Postman or Insomia or Thunder Client, create a new `POST /stkpush` request to your ngrok url with the following parameters.

```json
{
  "phoneNumber": "2547xxxxxxxx",
  "amount": "1"
}
```

- When request is sent, an STK Push Prompt is sent to phoneNumber provided above. The response should look like this.

```json
[
  "success",
  {
    "MerchantRequestID": "xxxx-xxxx-xxxx-xxxx",
    "CheckoutRequestID": "ws_CO_XXXXXXXXXXXXXXXXXXXXXXXXXX",
    "ResponseCode": "0",
    "ResponseDescription": "Success. Request accepted for processing",
    "CustomerMessage": "Success. Request accepted for processing"
  }
]
```

#### STK push query request

- After one has paid, you can use the mpesa query api to check if the payment was successful or not
- On APIs ---> M-pesa Express you can simulate a [query](https://developer.safaricom.co.ke/APIs/MpesaExpressSimulate), a stk quesry push request by selecting your app and inputing the CheckoutRequestID you got from the previous step.

```
  payload = {
    BusinessShortCode --> The organization shortcode used to receive the transaction.
    Password --> Should be encoded with base64 format (business_short_code + mpesa_passkey+timestamp).
    Timestamp --> The timestamp of the transaction in the format ???%Y%m%d%H%M%S???
    CheckoutRequestID --> The CheckoutRequestID used to identify the m-pesa transaction.
  }
```

- Add the following code to `app/controllers/mpesas_controller.rb`.

```ruby
  def stkquery
        url = "https://sandbox.safaricom.co.ke/mpesa/stkpushquery/v1/query"

        timestamp = "#{Time.now.strftime "%Y%m%d%H%M%S"}"
        business_short_code = ENV["MPESA_SHORTCODE"]
        password = Base64.strict_encode64("#{business_short_code}#{ENV["MPESA_PASSKEY"]}#{timestamp}")
        payload = {
            'BusinessShortCode': business_short_code,
            'Password': password,
            'Timestamp': timestamp,
             # Check if payment has been paid
            'CheckoutRequestID': params[:checkoutRequestID]
        }.to_json

        headers = {
            Content_type: 'application/json',
            Authorization: "Bearer #{ get_access_token }"
        }

        response = RestClient::Request.new({
            method: :post,
            url: url,
            payload: payload,
            headers: headers
            }).execute do |response, request|
                case response.code
                when 500
                    [ :error, JSON.parse(response.to_str) ]
                when 400
                    [ :error, JSON.parse(response.to_str) ]
                when 200
                    [ :success, JSON.parse(response.to_str) ]
                else
                    fail "Invalid response #{response.to_str} received."
                end
            end
        render json: response


    end
```

- Navigate to `config/routes.rb` and add this code.

```ruby
post 'stkquery', to: 'mpesas#stkquery'
```

- Open up Postman or Insomia or Thunder Client, create a new `POST /stkquery` request to your ngrok url with the following parameters.

```json
{
  "checkoutRequestID": "ws_CO_XXXXXXXXXXXXXXXXXXXXXXXXXX"
}
```

- Response looks like this:

```json
[
  "success",
  {
    "ResponseCode": "0",
    "ResponseDescription": "The service request has been accepted successsfully",
    "MerchantRequestID": "xxxx-xxxx-xxxxxxxxx-x",
    "CheckoutRequestID": "ws_CO_XXXXXXXXXXXXXXXXXXXXXXXXXX",
    "ResultCode": "0",
    "ResultDesc": "The service request is processed successfully."
  }
]
```

> You can use ResultDesc as a message prompt for your Client.

- The resulting Full Code on `app/controllers/mpesas_controller.rb`.

```ruby
class MpesasController < ApplicationController

    require 'rest-client'

    # stkpush
    # make payment

    def stkpush
        phoneNumber = params[:phoneNumber]
        amount = params[:amount]

        url = "https://sandbox.safaricom.co.ke/mpesa/stkpush/v1/processrequest"
        timestamp = "#{Time.now.strftime "%Y%m%d%H%M%S"}"
        business_short_code = ENV["MPESA_SHORTCODE"]
        password = Base64.strict_encode64("#{business_short_code}#{ENV["MPESA_PASSKEY"]}#{timestamp}")
        payload = {
            'BusinessShortCode': business_short_code,
            'Password': password,
            'Timestamp': timestamp,
            'TransactionType': "CustomerPayBillOnline",
            'Amount': amount,
            'PartyA': phoneNumber,
            'PartyB': business_short_code,
            'PhoneNumber': phoneNumber,
            'CallBackURL': "#{ENV["CALLBACK_URL"]}/callback_url",
            'AccountReference': 'Mpesa_ROR',
            'TransactionDesc': "Payment for Mpesa_ROR"
        }.to_json

        headers = {
            Content_type: 'application/json',
            Authorization: "Bearer #{get_access_token}"
        }

        response = RestClient::Request.new({
            method: :post,
            url: url,
            payload: payload,
            headers: headers
            }).execute do |response, request|
                case response.code
                when 500
                    [ :error, JSON.parse(response.to_str) ]
                when 400
                    [ :error, JSON.parse(response.to_str) ]
                when 200
                    [ :success, JSON.parse(response.to_str) ]
                else
                    fail "Invalid response #{response.to_str} received."
                end
            end
        render json: response


    end


    # stkquery
    # confirm if payment is gone through

    def stkquery
        url = "https://sandbox.safaricom.co.ke/mpesa/stkpushquery/v1/query"

        timestamp = "#{Time.now.strftime "%Y%m%d%H%M%S"}"
        business_short_code = ENV["MPESA_SHORTCODE"]
        password = Base64.strict_encode64("#{business_short_code}#{ENV["MPESA_PASSKEY"]}#{timestamp}")
        payload = {
            'BusinessShortCode': business_short_code,
            'Password': password,
            'Timestamp': timestamp,
             # Check if payment has been paid
            'CheckoutRequestID': params[:checkoutRequestID]
        }.to_json

        headers = {
            Content_type: 'application/json',
            Authorization: "Bearer #{ get_access_token }"
        }

        response = RestClient::Request.new({
            method: :post,
            url: url,
            payload: payload,
            headers: headers
            }).execute do |response, request|
                case response.code
                when 500
                    [ :error, JSON.parse(response.to_str) ]
                when 400
                    [ :error, JSON.parse(response.to_str) ]
                when 200
                    [ :success, JSON.parse(response.to_str) ]
                else
                    fail "Invalid response #{response.to_str} received."
                end
            end
        render json: response


    end

    private

    def generate_access_token_request
        @url = "https://sandbox.safaricom.co.ke/oauth/v1/generate?grant_type=client_credentials"
        @consumer_key = ENV['MPESA_CONSUMER_KEY']
        @consumer_secret = ENV['MPESA_CONSUMER_SECRET']
        @userpass = Base64::strict_encode64("#{@consumer_key}:#{@consumer_secret}")
        headers = {
            Authorization: "Bearer #{@userpass}"
        }
        res = RestClient::Request.execute( url: @url, method: :get, headers: {
            Authorization: "Basic #{@userpass}"
        })
        res
    end

    def get_access_token

        res = generate_access_token_request()
        if res.code != 200
            r = generate_access_token_request()
            if res.code != 200
                raise MpesaError('Unable to generate access token')
            end
        end

        body = JSON.parse(res, { symbolize_names: true })
        token = body[:access_token]
        AccessToken.destroy_all()
        AccessToken.create!(token: token)
        token
    end

end
```

- Your `config/routes.rb` should look like this.

```ruby
Rails.application.routes.draw do
  resources :mpesas
  post 'stkpush', to: 'mpesas#stkpush'
  post 'stkquery', to: 'mpesas#stkquery'
end
```

- The full code can be accessed [here](https://github.com/Felix-Barosio/Mpesa-ROR) from my repository.

> **N/B** - Remember to add the your local_env.yml file in .gitignore.

### Author Info

- [Felix Barosio](https://github.com/Felix-Barosio) ~ [Email](barosiofelix@gmail.com)
