# Setup for M-pesa stkpush and stkpushquery on Rails APIs.

### Daraja Safaricom Developers Portal ~ Setup

- Get started by login in or create [Daraja](https://developer.safaricom.co.ke/) Developer free Account.
- On apps tab, create a new sandbox app and give it your preferd name. Proceed to tick all the check boxes and click on create app.
- You will be redirected to the app details page where you will find your consumer key and consumer secret. This will be crucial later during setup.
- Navigate to the APIs tab and on M-pesa Express click on Simulate, on the input prompt select the app you just created.
- Scroll down and click on test credentials. The initiator password and passkey will crucial also later.

### Ngrok ~ Setup

- Login or create [ngrok](https://ngrok.com/) free account.
- Install on ubuntu by `sudo snap install ngrok` or download from website.
- Connect your account to ngrok run `ngrok authtoken <your authtoken>`.

### Rails ~ Setup

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

> **Be sure to replace origins 'example.com' with origins '\*'if you uncomment existing code instead of adding the above code.**

### Author Info

- [Felix Barosio](https://github.com/Felix-Barosio) ~ [Email](barosiofelix@gmail.com)
