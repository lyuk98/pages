---
tags:
  - Blog
  - Docker
  - Google Cloud Platform
  - Listed
tocopen: true
date: '2025-04-21'
title: Using custom domain with Listed
description: I wanted to use a custom domain for my blog. It was supposed to be easy, but the steps I took to solve the problem during the integration were far more than what I have initially expected.
---

Since around September 2025, Listed [no longer accepts](https://github.com/standardnotes/listed/pull/302 "chore: remove new custom domain from settings by antsgar · Pull Request #302 · standardnotes/listed") new requests for custom domains. The information below is therefore no longer relevant, but nevertheless kept as a record of what I have been through to make it work back then.

---

> Custom domains are available for Standard Notes members with an active Productivity or Professional [plan](https://standardnotes.com/plans). Domains include an HTTPS certificate, and require only a simple DNS record on your end.
> 
> Before submitting this form, please create an "A" record with your DNS provider with value 18.205.249.107.

Using a custom domain for my Listed blog is supposedly very easy. As I was instructed, I went to my DNS provider to add an `A` record.

![A form at Cloudflare Dashboard for adding a DNS record for my domain. The type is set to A. The name, which is a required field, is set to @. The IPv4 address, which is also a required field, is set to 18.205.249.107. The proxy status is set to DNS only. TTL is set to Auto.](https://images.lyuk98.com/5f8f7fb6-8c18-44a9-a1b6-d8aaf8bb37ef.avif "Adding a new A record")

I then went to my blog's settings page and filled up the form for custom domain.

![A section of the blog's settings titled Custom domain. There are two blank fields which are labelled "Standard Notes account email address" and "Your domain", respectively. The Submit button is disabled. The description is as follows: "Custom domains are available for Standard Notes members with an active Productivity or Professional plan. Domains include an HTTPS certificate, and require only a simple DNS record on your end. Before submitting this form, please create an A record with your DNS provider with value 18.205.249.107."](https://images.lyuk98.com/ff626e70-e35d-45b6-b9e1-6d8701d4f657.avif "Submitting a custom domain request")

The request for a custom domain was made. After a while, however, I received a disappointing email.

---

# The problem

> Hi there, your Listed domain settings were not configured properly. Please make sure you have an A record pointing to **18.205.249.107**, then **submit your domain request again via your author settings**. You'll know you have it configured correctly if when you visit your custom domain, it shows an Invalid Certificate error.
> 
> Any questions? Please feel free to reply directly to this email.

As mentioned in [my first post](https://lyuk98.com/60208/getting-started-with-listed "Getting started with Listed") in this blog, I was not able to link my custom domain with Listed. I have seen someone saying disabling DNSSEC helped them, but it was not the case for me. Since I did not want to email someone unless it was absolutely necessary, I decided to find the cause myself and see if there is anything I can do.

This is not meant to be a how-to guide on solving the problem, but rather a journey on how I reached that conclusion.

---

# Reading the source

## Listed

[Source](https://github.com/standardnotes/listed "standardnotes/listed: Create an online publication with automatic email newsletters. https://listed.to") for Listed is available, so I started from there. To find text across the project easier, I cloned the repository and opened the directory with [Code](https://github.com/microsoft/vscode "microsoft/vscode: Visual Studio Code").

```
[lyuk98@framework ~]$ git clone https://github.com/standardnotes/listed.git
[lyuk98@framework ~]$ code listed/
```

Searching for `custom domain` brought me to the frontend component [`CustomDomain.jsx`](https://github.com/standardnotes/listed/tree/5750e416b82060a6058631a2d7fdfc301c3107f6/client/app/components/authors/settings/CustomDomain.jsx "listed/client/app/components/authors/settings/CustomDomain.jsx at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed") for custom domain registration. I then quickly found [the part of the code](https://github.com/standardnotes/listed/tree/5750e416b82060a6058631a2d7fdfc301c3107f6/client/app/components/authors/settings/CustomDomain.jsx#L38 "listed/client/app/components/authors/settings/CustomDomain.jsx at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed") that was responsible for making requests.

```jsx
const response = await axios
    .post(`/authors/${author.id}/domain_request?secret=${author.secret}`, null, {
        headers: {
            "X-CSRF-Token": getAuthToken(),
        },
        data: {
            extended_email: extendedEmail,
            domain,
        },
    });
```

Following where the request goes to, I located [the controller](https://github.com/standardnotes/listed/tree/5750e416b82060a6058631a2d7fdfc301c3107f6/app/controllers/authors_controller.rb#L352 "listed/app/controllers/authors_controller.rb at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed") in [`authors_controller.rb`](https://github.com/standardnotes/listed/tree/5750e416b82060a6058631a2d7fdfc301c3107f6/app/controllers/authors_controller.rb "listed/app/controllers/authors_controller.rb at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed") by searching `domain_request`.

```ruby
def domain_request
  existing_domain = Domain.find_by_domain(params[:domain])
  if existing_domain
    render :json => { message: "Domain #{params[:domain]} is already taken." }, :status => :conflict

    return
  end

  if !@author.domain
    @author.domain = Domain.new
  end

  @author.domain.domain = params[:domain]
  @author.domain.extended_email = params[:extended_email]
  @author.domain.approved = false
  @author.domain.active = false
  @author.domain.save

  SslCertificateCreateJob.perform_later(params[:domain])

  redirect_to_authenticated_settings(@author)
end
```

I was surprised to come across Ruby, as I had no prior experience with it. Nonetheless, I focused on [the part](https://github.com/standardnotes/listed/tree/5750e416b82060a6058631a2d7fdfc301c3107f6/app/controllers/authors_controller.rb#L370 "listed/app/controllers/authors_controller.rb at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed") that seemed to request creation of an SSL certificate for the domain.

```ruby
SslCertificateCreateJob.perform_later(params[:domain])
```

To see what it does, I searched for `SslCertificateCreateJob` and found a short class definition at [`ssl_certificate_create_job.rb`](https://github.com/standardnotes/listed/tree/5750e416b82060a6058631a2d7fdfc301c3107f6/app/jobs/ssl_certificate_create_job.rb "listed/app/jobs/ssl_certificate_create_job.rb at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed").

```ruby
class SslCertificateCreateJob < ApplicationJob
  def perform(domain)
    SSLCertificate.find_or_create_by(domain: domain)
  end
end
```

Curious about what `SSLCertificate` does, I found its definition at [`ssl_certificate.rb`](https://github.com/standardnotes/listed/tree/5750e416b82060a6058631a2d7fdfc301c3107f6/app/models/ssl_certificate.rb "listed/app/models/ssl_certificate.rb at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed"). Although I could not find references to `find_or_create_by` there (I later learned that it is something about models), [the part](https://github.com/standardnotes/listed/tree/5750e416b82060a6058631a2d7fdfc301c3107f6/app/models/ssl_certificate.rb#L25 "listed/app/models/ssl_certificate.rb at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed") that apparently does the domain validation was present.

```ruby
# rails-letsencrypt does not return error value on `verify` method, so we can't differentiate
# between a rate limiting error and an invalid domain. Use this method to custom validate
# whether a domain's DNS records are correctly configured.
# Returns 'valid' if valid, 'invalid' if explicitly invalid, and 'error' if API error.
# Based on https://github.com/elct9620/rails-letsencrypt/blob/master/app/models/concerns/lets_encrypt/certificate_verifiable.rb
def validate
  create_order
  start_challenge
  wait_verify_status
  status = @challenge.status
  return 'invalid' if status == 'invalid'

  'valid'
rescue StandardError => e
  Rails.logger.info "Error validating cerficate #{e.message}"
  'error'
end
```

Listed seemed to depend on [rails-letsencrypt](https://github.com/elct9620/rails-letsencrypt "elct9620/rails-letsencrypt: The Let's Encrypt certificate manager for rails") for HTTPS certificates. I decided to read its source, too.

## `rails-letsencrypt`

I first cloned the source of the Ruby project:

```
[lyuk98@framework ~]$ git clone https://github.com/elct9620/rails-letsencrypt.git
[lyuk98@framework ~]$ code rails-letsencrypt/
```

As the syntax suggested that `SSLCertificate` in Listed inherits `LetsEncrypt::Certificate`, I went to its definition at [`certificate.rb`](https://github.com/elct9620/rails-letsencrypt/blob/715ea48650fd2244eab927288dc121c09afe3411/app/models/lets_encrypt/certificate.rb "rails-letsencrypt/app/models/lets_encrypt/certificate.rb at 715ea48650fd2244eab927288dc121c09afe3411 · elct9620/rails-letsencrypt"). I could not find references to the validation methods there, but by following the link the [validation method](https://github.com/standardnotes/listed/tree/5750e416b82060a6058631a2d7fdfc301c3107f6/app/models/ssl_certificate.rb#L25 "listed/app/models/ssl_certificate.rb at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed") mentioned, [one of the modules](https://github.com/elct9620/rails-letsencrypt/blob/715ea48650fd2244eab927288dc121c09afe3411/app/models/concerns/lets_encrypt/certificate_verifiable.rb "rails-letsencrypt/app/models/concerns/lets_encrypt/certificate_verifiable.rb at 715ea48650fd2244eab927288dc121c09afe3411 · elct9620/rails-letsencrypt") that `LetsEncrypt::Certificate` includes was located. From there, I found [the part](https://github.com/elct9620/rails-letsencrypt/blob/715ea48650fd2244eab927288dc121c09afe3411/app/models/concerns/lets_encrypt/certificate_verifiable.rb#L28 "rails-letsencrypt/app/models/concerns/lets_encrypt/certificate_verifiable.rb at 715ea48650fd2244eab927288dc121c09afe3411 · elct9620/rails-letsencrypt") that apparently does the domain validation:

```ruby
def start_challenge
  logger.info "Attempting verification of #{domain}"
  @challenge.request_validation
end
```

I tried searching for `request_validation`, but I could not find its definition. However, initialisation of `@challenge` could be seen at [the method](https://github.com/elct9620/rails-letsencrypt/blob/715ea48650fd2244eab927288dc121c09afe3411/app/models/concerns/lets_encrypt/certificate_verifiable.rb#L20 "rails-letsencrypt/app/models/concerns/lets_encrypt/certificate_verifiable.rb at 715ea48650fd2244eab927288dc121c09afe3411 · elct9620/rails-letsencrypt") `create_order`.

```ruby
def create_order
  # TODO: Support multiple domain
  @challenge = order.authorizations.first.http
  self.verification_path = @challenge.filename
  self.verification_string = @challenge.file_content
  save!
end
```

Searching for `order` led me back to [the class](https://github.com/elct9620/rails-letsencrypt/blob/715ea48650fd2244eab927288dc121c09afe3411/app/models/lets_encrypt/certificate.rb "rails-letsencrypt/app/models/lets_encrypt/certificate.rb at 715ea48650fd2244eab927288dc121c09afe3411 · elct9620/rails-letsencrypt") that included the module, with [its definition](https://github.com/elct9620/rails-letsencrypt/blob/715ea48650fd2244eab927288dc121c09afe3411/app/models/lets_encrypt/certificate.rb#L92 "rails-letsencrypt/app/models/lets_encrypt/certificate.rb at 715ea48650fd2244eab927288dc121c09afe3411 · elct9620/rails-letsencrypt") inside.

```ruby
def order
  @order ||= LetsEncrypt.client.new_order(identifiers: [domain])
end
```

I then searched for `def client`, which led to [another definition](https://github.com/elct9620/rails-letsencrypt/blob/715ea48650fd2244eab927288dc121c09afe3411/lib/letsencrypt.rb#L23 "rails-letsencrypt/lib/letsencrypt.rb at 715ea48650fd2244eab927288dc121c09afe3411 · elct9620/rails-letsencrypt").


```ruby
# Create the ACME Client to Let's Encrypt
def client
  @client ||= ::Acme::Client.new(
    private_key: private_key,
    directory: directory
  )
end
```

[Acme::Client](https://github.com/unixcharles/acme-client "unixcharles/acme-client: A Ruby client for the letsencrypt's ACME protocol.") looked like a dependency of this project. I prepared myself for reading another project.

## Acme::Client

Just like the previous two repositories, I cloned [the repository](https://github.com/unixcharles/acme-client "unixcharles/acme-client: A Ruby client for the letsencrypt's ACME protocol."). Since the version of `acme-client` that `rails-letsencrypt` mentioned in the lockfile was 2.0.15, I switched to the commit at that point.

```
[lyuk98@framework ~]$ git clone https://github.com/unixcharles/acme-client.git
[lyuk98@framework ~]$ cd acme-client/
[lyuk98@framework acme-client]$ git switch --detach v2.0.15
[lyuk98@framework acme-client]$ code .
```

I first read [its documentation](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/README.md "acme-client/README.md at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client"), which gave me a rough idea about the process.

1. Setting up a client
2. Creating an order with the client
3. Accessing the HTTP-01 challenge
4. Completing the challenge

### Setting up a client

Looking back at `rails-letsencrypt` again, especially [the client definition](https://github.com/elct9620/rails-letsencrypt/blob/715ea48650fd2244eab927288dc121c09afe3411/lib/letsencrypt.rb "rails-letsencrypt/lib/letsencrypt.rb at 715ea48650fd2244eab927288dc121c09afe3411 · elct9620/rails-letsencrypt"), I could now see that a new client is being made with a private key and a directory of `https://acme-v02.api.letsencrypt.org/directory`.

```ruby
module LetsEncrypt
  # Production mode API Endpoint
  ENDPOINT = 'https://acme-v02.api.letsencrypt.org/directory'
  
  # ...

  class << self
    # Create the ACME Client to Let's Encrypt
    def client
      @client ||= ::Acme::Client.new(
        private_key: private_key,
        directory: directory
      )
    end

    def private_key
      @private_key ||= OpenSSL::PKey::RSA.new(load_private_key)
    end

    def load_private_key
      return ENV.fetch('LETSENCRYPT_PRIVATE_KEY', nil) if config.use_env_key
      return File.open(private_key_path) if File.exist?(private_key_path)

      generate_private_key
    end

    # Get current using Let's Encrypt endpoint
    def directory
      @directory ||= config.use_staging? ? ENDPOINT_STAGING : ENDPOINT
    end

    # ...

    def private_key_path
      config.private_key_path || Rails.root.join('config/letsencrypt.key')
    end

    def generate_private_key
      key = OpenSSL::PKey::RSA.new(4096)
      File.write(private_key_path, key.to_s)
      logger.info "Created new private key for Let's Encrypt"
      key
    end
    
    # ...

  end
end
```

How it obtains a private key was unclear at first, but I later found that Listed sets an environment variable `LETSENCRYPT_PRIVATE_KEY` with [its dotenv configuration](https://github.com/standardnotes/listed/blob/5750e416b82060a6058631a2d7fdfc301c3107f6/.env.sample#L21 "listed/.env.sample at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed").

```
# Custom Domains
LETSENCRYPT_PRIVATE_KEY=
CUSTOM_DOMAIN_IP=
```

### Creating an order with the client

Looking back at what `rails-letsencrypt` did, how it [creates orders](https://github.com/elct9620/rails-letsencrypt/blob/715ea48650fd2244eab927288dc121c09afe3411/app/models/lets_encrypt/certificate.rb#L92 "rails-letsencrypt/app/models/lets_encrypt/certificate.rb at 715ea48650fd2244eab927288dc121c09afe3411 · elct9620/rails-letsencrypt") was pretty straightforward.

```ruby
def order
  @order ||= LetsEncrypt.client.new_order(identifiers: [domain])
end
```

However, how it works under the hood was not. Looking at [the method](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client.rb#L138 "acme-client/lib/acme/client.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client") at [`client.rb`](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client.rb "acme-client/lib/acme/client.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client"), it was apparently making a POST request somewhere.

```ruby
def new_order(identifiers:, not_before: nil, not_after: nil)
  payload = {}
  payload['identifiers'] = prepare_order_identifiers(identifiers)
  payload['notBefore'] = not_before if not_before
  payload['notAfter'] = not_after if not_after

  response = post(endpoint_for(:new_order), payload: payload)
  arguments = attributes_from_order_response(response)
  Acme::Client::Resources::Order.new(self, **arguments)
end
```

The `endpoint_for` [method](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client.rb#L353 "acme-client/lib/acme/client.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client") calls the method of the same name from the `@directory` object, [which is initialised](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client.rb#L47 "acme-client/lib/acme/client.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client") with given options.

```ruby
class Acme::Client
  DEFAULT_DIRECTORY = 'http://127.0.0.1:4000/directory'.freeze
  # ...

  def initialize(jwk: nil, kid: nil, private_key: nil, directory: DEFAULT_DIRECTORY, connection_options: {}, bad_nonce_retry: 0)
    # ...
    @directory = Acme::Client::Resources::Directory.new(URI(directory), @connection_options)
    @nonces ||= []
  end

  # ...

  def endpoint_for(key)
    @directory.endpoint_for(key)
  end
end
```

As seen [earlier](https://github.com/elct9620/rails-letsencrypt/blob/715ea48650fd2244eab927288dc121c09afe3411/lib/letsencrypt.rb#L23 "rails-letsencrypt/lib/letsencrypt.rb at 715ea48650fd2244eab927288dc121c09afe3411 · elct9620/rails-letsencrypt"), `rails-letsencrypt` sets custom values for `private_key` and `directory`, which are the environment variable `LETSENCRYPT_PRIVATE_KEY` and `https://acme-v02.api.letsencrypt.org/directory`, respectively.

I went to [the definition](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client/resources/directory.rb "acme-client/lib/acme/client/resources/directory.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client") of `Acme::Client::Resources::Directory`, and found following points of interest:

```ruby
class Acme::Client::Resources::Directory
  DIRECTORY_RESOURCES = {
    new_nonce: 'newNonce',
    new_account: 'newAccount',
    new_order: 'newOrder',
    new_authz: 'newAuthz',
    revoke_certificate: 'revokeCert',
    key_change: 'keyChange'
  }

  # ...

  def initialize(url, connection_options)
    @url, @connection_options = url, connection_options
  end

  def endpoint_for(key)
    directory.fetch(key) do |missing_key|
      raise Acme::Client::Error::UnsupportedOperation,
        "Directory at #{@url} does not include `#{missing_key}`"
    end
  end

  # ...

  private

  def directory
    @directory ||= load_directory
  end

  def load_directory
    body = fetch_directory
    result = {}
    result[:meta] = body.delete('meta')
    DIRECTORY_RESOURCES.each do |key, entry|
      result[key] = URI(body[entry]) if body[entry]
    end
    result
  rescue JSON::ParserError => exception
    raise Acme::Client::Error::InvalidDirectory,
      "Invalid directory url\n#{@directory} did not return a valid directory\n#{exception.inspect}"
  end

  def fetch_directory
    http_client = Acme::Client::HTTPClient.new_acme_connection(url: @directory, options: @connection_options, client: nil, mode: nil)
    response = http_client.get(@url)
    response.body
  end
end
```

I could see that with `initialize`, the object would be set up with the given URL and an empty `connection_options`. However, I became confused with how the `directory` is loaded.

1. `directory` tries to return `@directory`, but since it is unset, it would call `load_directory`.
2. `load_directory` would call `fetch_directory`.
3. `fetch_directory` would create a new connection, with... `@directory`? I thought it was not set yet...

[The method](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client/http_client.rb#L9 "acme-client/lib/acme/client/http_client.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client") `new_acme_connection` apparently returns an instance of `Faraday::Connection`.

```ruby
# Creates and returns a new HTTP client designed for the Acme-protocol, with default settings.
#
# @param  url [URI:HTTPS]
# @param  client [Acme::Client]
# @param  mode [Symbol]
# @param  options [Hash]
# @param  bad_nonce_retry [Integer]
# @return [Faraday::Connection]
def self.new_acme_connection(url:, client:, mode:, options: {}, bad_nonce_retry: 0)
  new_connection(url: url, options: options) do |configuration|
    if bad_nonce_retry > 0
      configuration.request(:retry,
        max: bad_nonce_retry,
        methods: Faraday::Connection::METHODS,
        exceptions: [Acme::Client::Error::BadNonce])
    end

    configuration.use Acme::Client::HTTPClient::AcmeMiddleware, client: client, mode: mode

    yield(configuration) if block_given?
  end
end
```

Since the GET request is apparently made using the expected `@url` anyway, I briefly read [the definition](https://github.com/lostisland/faraday/blob/28a097f756fc10b13b7193e779c9bae5fec02fb3/lib/faraday/connection.rb "faraday/lib/faraday/connection.rb at 28a097f756fc10b13b7193e779c9bae5fec02fb3 · lostisland/faraday") of `Faraday::Connection` to confirm that I am not missing anything.

```ruby
class Connection
  # A Set of allowed HTTP verbs.
  METHODS = Set.new %i[get post put delete head patch options trace]
  USER_AGENT = "Faraday v#{VERSION}"

  # ...

  # Initializes a new Faraday::Connection.
  #
  # @param url [URI, String] URI or String base URL to use as a prefix for all
  #           requests (optional).
  # @param options [Hash, Faraday::ConnectionOptions]
  # @option options [URI, String] :url ('http:/') URI or String base URL
  # @option options [Hash<String => String>] :params URI query unencoded
  #                 key/value pairs.
  # @option options [Hash<String => String>] :headers Hash of unencoded HTTP
  #                 header key/value pairs.
  # @option options [Hash] :request Hash of request options.
  # @option options [Hash] :ssl Hash of SSL options.
  # @option options [Hash, URI, String] :proxy proxy options, either as a URL
  #                 or as a Hash
  # @option options [URI, String] :proxy[:uri]
  # @option options [String] :proxy[:user]
  # @option options [String] :proxy[:password]
  # @yield [self] after all setup has been done
  def initialize(url = nil, options = nil)
    # ...
  end

  # ...
end
```

The URL is optional and sending an unset `@directory` would probably lead to a same result, anyway.

Returning to Acme::Client at [`directory.rb`](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client/resources/directory.rb "acme-client/lib/acme/client/resources/directory.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client"), it was apparent that `load_directory` would do something with response for GET request. I tried making the request myself to see what kind of data it expects.

```
[lyuk98@framework ~]$ curl https://acme-v02.api.letsencrypt.org/directory
{
  "keyChange": "https://acme-v02.api.letsencrypt.org/acme/key-change",
  "l-bhbTubn0s": "https://community.letsencrypt.org/t/adding-random-entries-to-the-directory/33417",
  "meta": {
    "caaIdentities": [
      "letsencrypt.org"
    ],
    "profiles": {
      "classic": "https://letsencrypt.org/docs/profiles#classic",
      "shortlived": "https://letsencrypt.org/docs/profiles#shortlived (not yet generally available)",
      "tlsserver": "https://letsencrypt.org/docs/profiles#tlsserver (not yet generally available)"
    },
    "termsOfService": "https://letsencrypt.org/documents/LE-SA-v1.5-February-24-2025.pdf",
    "website": "https://letsencrypt.org"
  },
  "newAccount": "https://acme-v02.api.letsencrypt.org/acme/new-acct",
  "newNonce": "https://acme-v02.api.letsencrypt.org/acme/new-nonce",
  "newOrder": "https://acme-v02.api.letsencrypt.org/acme/new-order",
  "renewalInfo": "https://acme-v02.api.letsencrypt.org/draft-ietf-acme-ari-03/renewalInfo",
  "revokeCert": "https://acme-v02.api.letsencrypt.org/acme/revoke-cert"
}
```

What `load_directory` does looked like assigning URLs of supported operations into its own data structure. Since I am interested in `newOrder`, all I had to know was that the endpoint for making the POST request becomes the `URI` object of `https://acme-v02.api.letsencrypt.org/acme/new-order`.

I went back to [the method](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client.rb#L138 "acme-client/lib/acme/client.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client") `new_order` and saw how the payload for the request was being made.

```ruby
def new_order(identifiers:, not_before: nil, not_after: nil)
  payload = {}
  payload['identifiers'] = prepare_order_identifiers(identifiers)
  payload['notBefore'] = not_before if not_before
  payload['notAfter'] = not_after if not_after

  response = post(endpoint_for(:new_order), payload: payload)
  arguments = attributes_from_order_response(response)
  Acme::Client::Resources::Order.new(self, **arguments)
end
```

`rails-letsencrypt` does not set `not_before` and `not_after`, so I simply looked at `identifiers`. [The method](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client.rb#L254 "acme-client/lib/acme/client.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client") `prepare_order_identifiers` changes each element that is a `String` into a hash, which is then used as a request data.

```ruby
def prepare_order_identifiers(identifiers)
  if identifiers.is_a?(Hash)
    [identifiers]
  else
    Array(identifiers).map do |identifier|
      if identifier.is_a?(String)
        { type: 'dns', value: identifier }
      else
        identifier
      end
    end
  end
end
```

As far as I could tell at this point, it was making a request to `https://acme-v02.api.letsencrypt.org/acme/new-order`, with the data being something like the following:

```json
{
	"identifiers": [
		{
			"type": "dns",
			"value": "my domain"
		}
	]
}
```

I tried making the request myself to see what the response would be like.

```
[lyuk98@framework ~]$ curl --request POST \
--header 'Content-Type: application/json' \
--data '{"identifiers": [{"type": "dns","value": "my domain"}]}' \
https://acme-v02.api.letsencrypt.org/acme/new-order
{
  "type": "urn:ietf:params:acme:error:malformed",
  "detail": "Unable to validate JWS :: Invalid Content-Type header on POST. Content-Type must be \"application/jose+json\"",
  "status": 400
}
```

Okay, maybe I was a bit too impatient. Before trying to do something stupid any further, I decided to read [the documentation](https://datatracker.ietf.org/doc/html/rfc8555#section-7.4 "RFC 8555 - Automatic Certificate Management Environment (ACME)") and see the expected request and response formats.

> ```
> POST /acme/new-order HTTP/1.1
> Host: example.com
> Content-Type: application/jose+json
> 
> {
>   "protected": base64url({
>     "alg": "ES256",
>     "kid": "https://example.com/acme/acct/evOfKhNU60wg",
>     "nonce": "5XJ1L3lEkMG7tR6pA00clA",
>     "url": "https://example.com/acme/new-order"
>   }),
>   "payload": base64url({
>     "identifiers": [
>       { "type": "dns", "value": "www.example.org" },
>       { "type": "dns", "value": "example.org" }
>     ],
>     "notBefore": "2016-01-01T00:04:00+04:00",
>     "notAfter": "2016-01-08T00:04:00+04:00"
>   }),
>   "signature": "H6ZXtGjTZyUnPeKn...wEA4TklBdh3e454g"
> }
> ```

> ```
> HTTP/1.1 201 Created
> Replay-Nonce: MYAuvOpaoIiywTezizk5vw
> Link: <https://example.com/acme/directory>;rel="index"
> Location: https://example.com/acme/order/TOlocE8rfgo
> 
> {
>   "status": "pending",
>   "expires": "2016-01-05T14:09:07.99Z",
> 
>   "notBefore": "2016-01-01T00:00:00Z",
>   "notAfter": "2016-01-08T00:00:00Z",
> 
>   "identifiers": [
>     { "type": "dns", "value": "www.example.org" },
>     { "type": "dns", "value": "example.org" }
>   ],
> 
>   "authorizations": [
>     "https://example.com/acme/authz/PAniVnsZcis",
>     "https://example.com/acme/authz/r4HqLzrSrpI"
>   ],
> 
>   "finalize": "https://example.com/acme/order/TOlocE8rfgo/finalize"
> }
> ```

With the response now present, `arguments` [would be set](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client.rb#L145 "acme-client/lib/acme/client.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client"), which is in turn used to create a new `Acme::Client::Resources::Order`. `attributes_from_order_response` apparently also looks for the key `certificate`, which was not present in [the specification](https://datatracker.ietf.org/doc/html/rfc8555#section-7.4 "RFC 8555 - Automatic Certificate Management Environment (ACME)"), but I thought it was probably insignificant and moved on.

```ruby
def new_order(identifiers:, not_before: nil, not_after: nil)
  payload = {}
  payload['identifiers'] = prepare_order_identifiers(identifiers)
  payload['notBefore'] = not_before if not_before
  payload['notAfter'] = not_after if not_after

  response = post(endpoint_for(:new_order), payload: payload)
  arguments = attributes_from_order_response(response)
  Acme::Client::Resources::Order.new(self, **arguments)
end

# ...

def attributes_from_order_response(response)
  attributes = extract_attributes(
    response.body,
    :status,
    :expires,
    [:finalize_url, 'finalize'],
    [:authorization_urls, 'authorizations'],
    [:certificate_url, 'certificate'],
    :identifiers
  )

  attributes[:url] = response.headers[:location] if response.headers[:location]
  attributes
end

# ...

def extract_attributes(input, *attributes)
  attributes
    .map {|fields| Array(fields) }
    .each_with_object({}) { |(key, field), hash|
    field ||= key.to_s
    hash[key] = input[field]
  }
end
```

The `order` object [would then be created](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client/resources/order.rb#L6 "acme-client/lib/acme/client/resources/order.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client") with `status`, `expires`, `finalize_url` (from `finalize`), `authorization_urls` (from `authorizations`), `certificate_url` (from `certificate`), and `identifiers`.

```ruby
class Acme::Client::Resources::Order
  attr_reader :url, :status, :contact, :finalize_url, :identifiers, :authorization_urls, :expires, :certificate_url

  def initialize(client, **arguments)
    @client = client
    assign_attributes(**arguments)
  end

  # ...

  private

  def assign_attributes(url:, status:, expires:, finalize_url:, authorization_urls:, identifiers:, certificate_url: nil)
    @url = url
    @status = status
    @expires = expires
    @finalize_url = finalize_url
    @authorization_urls = authorization_urls
    @identifiers = identifiers
    @certificate_url = certificate_url
  end
end
```

### Accessing the HTTP-01 challenge

To figure out how a challenge is created, I returned to `rails-letsencrypt` once again, especially `create_order` at [`certificate_verifiable.rb`](https://github.com/elct9620/rails-letsencrypt/blob/715ea48650fd2244eab927288dc121c09afe3411/app/models/concerns/lets_encrypt/certificate_verifiable.rb#L20 "rails-letsencrypt/app/models/concerns/lets_encrypt/certificate_verifiable.rb at 715ea48650fd2244eab927288dc121c09afe3411 · elct9620/rails-letsencrypt").

```ruby
def create_order
  # TODO: Support multiple domain
  @challenge = order.authorizations.first.http
  self.verification_path = @challenge.filename
  self.verification_string = @challenge.file_content
  save!
end
```

At `Acme::Client::Resources::Order`, `authorizations` [would call](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client/resources/order.rb#L16 "acme-client/lib/acme/client/resources/order.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client") `@client`'s `authorization` for each URL from `@authorization_urls`.

```ruby
def authorizations
  @authorization_urls.map do |authorization_url|
    @client.authorization(url: authorization_url)
  end
end
```

The client would then... make another (POST-as-GET) request? It seemed like there are many requests involved (which I later learned [in detail](https://datatracker.ietf.org/doc/html/rfc8555#section-7.1 "RFC 8555 - Automatic Certificate Management Environment (ACME)")).

```ruby
def authorization(url:)
  response = post_as_get(url)
  arguments = attributes_from_authorization_response(response)
  Acme::Client::Resources::Authorization.new(self, url: url, **arguments)
end
```

I paid attention to [the response data for the order creation](https://datatracker.ietf.org/doc/html/rfc8555#section-7.4 "RFC 8555 - Automatic Certificate Management Environment (ACME)"). The value with the key `authorizations` is expected to be an array of URLs, so I thought they are where the next requests reach.

> ```
> HTTP/1.1 201 Created
> Replay-Nonce: MYAuvOpaoIiywTezizk5vw
> Link: <https://example.com/acme/directory>;rel="index"
> Location: https://example.com/acme/order/TOlocE8rfgo
> 
> {
>   "status": "pending",
>   "expires": "2016-01-05T14:09:07.99Z",
> 
>   "notBefore": "2016-01-01T00:00:00Z",
>   "notAfter": "2016-01-08T00:00:00Z",
> 
>   "identifiers": [
>     { "type": "dns", "value": "www.example.org" },
>     { "type": "dns", "value": "example.org" }
>   ],
> 
>   "authorizations": [
>     "https://example.com/acme/authz/PAniVnsZcis",
>     "https://example.com/acme/authz/r4HqLzrSrpI"
>   ],
> 
>   "finalize": "https://example.com/acme/order/TOlocE8rfgo/finalize"
> }
> ```

Within the same specification, I found [the part that documents](https://datatracker.ietf.org/doc/html/rfc8555#section-7.5 "RFC 8555 - Automatic Certificate Management Environment (ACME)") the request and response format for "Identifier Authorization", having the endpoint of `/acme/authz/[resource]`.

> ```
> POST /acme/authz/PAniVnsZcis HTTP/1.1
> Host: example.com
> Content-Type: application/jose+json
> 
> {
>   "protected": base64url({
>     "alg": "ES256",
>     "kid": "https://example.com/acme/acct/evOfKhNU60wg",
>     "nonce": "uQpSjlRb4vQVCjVYAyyUWg",
>     "url": "https://example.com/acme/authz/PAniVnsZcis"
>   }),
>   "payload": "",
>   "signature": "nuSDISbWG8mMgE7H...QyVUL68yzf3Zawps"
> }
> ```
> 
> ```
> HTTP/1.1 200 OK
> Content-Type: application/json
> Link: <https://example.com/acme/directory>;rel="index"
> 
> {
>   "status": "pending",
>   "expires": "2016-01-02T14:09:30Z",
> 
>   "identifier": {
>     "type": "dns",
>     "value": "www.example.org"
>   },
> 
>   "challenges": [
>     {
>       "type": "http-01",
>       "url": "https://example.com/acme/chall/prV_B7yEyA4",
>       "token": "DGyRejmCefe7v4NfDGDKfA"
>     },
>     {
>       "type": "dns-01",
>       "url": "https://example.com/acme/chall/Rg5dV14Gh1Q",
>       "token": "DGyRejmCefe7v4NfDGDKfA"
>     }
>   ]
> }
> ```

After crafting `arguments` from the response data (using `attributes_from_authorization_response`), a new instance of `Acme::Client::Resources::Authorization` [would be created](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client/resources/authorization.rb#L6 "acme-client/lib/acme/client/resources/authorization.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client").

```ruby
def initialize(client, **arguments)
  @client = client
  assign_attributes(**arguments)
end

# ...

def assign_attributes(url:, status:, expires:, challenges:, identifier:, wildcard: false)
  @url = url
  @identifier = identifier
  @domain = identifier.fetch('value')
  @status = status
  @expires = expires
  @challenges = challenges
  @wildcard = wildcard
end
```

`rails-letsencrypt` does the HTTP-01 challenge to obtain a certificate. I first read [what it is](https://letsencrypt.org/docs/challenge-types/#http-01-challenge "Challenge Types -  Let's Encrypt") and what needs to be done to pass the challenge.

> Let’s Encrypt gives a token to your ACME client, and your ACME client puts a file on your web server at `http://<YOUR_DOMAIN>/.well-known/acme-challenge/<TOKEN>`.

The object responsible for HTTP-01 validation was being returned from the method `http01` at [`authorization.rb`](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client/resources/authorization.rb#L27 "acme-client/lib/acme/client/resources/authorization.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client"), which could also be called as `http`.

```ruby
def http01
  @http01 ||= challenges.find { |challenge|
    challenge.is_a?(Acme::Client::Resources::Challenges::HTTP01)
  }
end
alias_method :http, :http01
```

[The method](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client/resources/authorization.rb#L21 "acme-client/lib/acme/client/resources/authorization.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client") `challenges` internally calls `initialize_challenge`, which internally creates new instances of `Acme::Client::Resources::Challenges` based on the previous response data.

```ruby
def challenges
  @challenges.map do |challenge|
    initialize_challenge(challenge)
  end
end

# ...

def initialize_challenge(attributes)
  arguments = {
    type: attributes.fetch('type'),
    status: attributes.fetch('status'),
    url: attributes.fetch('url'),
    token: attributes.fetch('token'),
    error: attributes['error']
  }
  Acme::Client::Resources::Challenges.new(@client, **arguments)
end
```

An instance representing a challenge, which type is dependent on the parameter `type`, is then [created](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client/resources/challenges.rb#L14 "acme-client/lib/acme/client/resources/challenges.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client").

```ruby
module Acme::Client::Resources::Challenges
  require 'acme/client/resources/challenges/base'
  require 'acme/client/resources/challenges/http01'
  require 'acme/client/resources/challenges/dns01'
  require 'acme/client/resources/challenges/unsupported_challenge'

  CHALLENGE_TYPES = {
    'http-01' => Acme::Client::Resources::Challenges::HTTP01,
    'dns-01' => Acme::Client::Resources::Challenges::DNS01
  }

  def self.new(client, type:, **arguments)
    CHALLENGE_TYPES.fetch(type, Unsupported).new(client, **arguments)
  end
end
```

Since the requested challenge was `http-01`, an instance of `Acme::Client::Resources::Challenges::HTTP01` would be returned.

### Completing the challenge

Going back to `rails-letsencrypt` yet again, at [the method](https://github.com/elct9620/rails-letsencrypt/blob/715ea48650fd2244eab927288dc121c09afe3411/app/models/concerns/lets_encrypt/certificate_verifiable.rb#L20 "rails-letsencrypt/app/models/concerns/lets_encrypt/certificate_verifiable.rb at 715ea48650fd2244eab927288dc121c09afe3411 · elct9620/rails-letsencrypt") `create_order`, `@challenge`'s `request_validation` is called. The derived class at [`http01.rb`](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client/resources/challenges/http01.rb "acme-client/lib/acme/client/resources/challenges/http01.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client") did not have the method I was looking for, but the base class at [`base.rb`](https://github.com/unixcharles/acme-client/blob/c819649962d66bb6bcc947416c5c57d482b0693f/lib/acme/client/resources/challenges/base.rb "acme-client/lib/acme/client/resources/challenges/base.rb at c819649962d66bb6bcc947416c5c57d482b0693f · unixcharles/acme-client") did.

```ruby
def request_validation
  assign_attributes(**send_challenge_validation(
    url: url
  ))
  true
end

# ...

def send_challenge_validation(url:)
  @client.request_challenge_validation(
    url: url
  ).to_h
end

def assign_attributes(status:, url:, token:, error: nil)
  @status = status
  @url = url
  @token = token
  @error = error
end
```

It was apparently making a request using `@client` and expecting a response with keys `status`, `url`, `token`, and `error`. Instead of reading the client's code, I [read the specification and found](https://datatracker.ietf.org/doc/html/rfc8555#section-7.5.1 "RFC 8555 - Automatic Certificate Management Environment (ACME)") where the request data format for responding to challenges are documented.

> ```
> POST /acme/chall/prV_B7yEyA4 HTTP/1.1
> Host: example.com
> Content-Type: application/jose+json
> 
> {
>   "protected": base64url({
>     "alg": "ES256",
>     "kid": "https://example.com/acme/acct/evOfKhNU60wg",
>     "nonce": "Q_s3MWoqT05TrdkM2MTDcw",
>     "url": "https://example.com/acme/chall/prV_B7yEyA4"
>   }),
>   "payload": base64url({}),
>   "signature": "9cbg5JO1Gf5YLjjz...SpkUfcdPai9uVYYQ"
> }
> ```

What it said about the response was a bit underwhelming:

> The server provides a 200 (OK) response with the updated challenge object as its body.

It made sense, however, that Acme::Client would expect keys `status`, `url`, `token`, and `error`.

I returned to Listed's source, at [the validation part](https://github.com/standardnotes/listed/tree/5750e416b82060a6058631a2d7fdfc301c3107f6/app/models/ssl_certificate.rb#L25 "listed/app/models/ssl_certificate.rb at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed"). It checks if `@challenge.status` is `invalid`, and I assumed that was the case for my domain.

```ruby
# rails-letsencrypt does not return error value on `verify` method, so we can't differentiate
# between a rate limiting error and an invalid domain. Use this method to custom validate
# whether a domain's DNS records are correctly configured.
# Returns 'valid' if valid, 'invalid' if explicitly invalid, and 'error' if API error.
# Based on https://github.com/elct9620/rails-letsencrypt/blob/master/app/models/concerns/lets_encrypt/certificate_verifiable.rb
def validate
  create_order
  start_challenge
  wait_verify_status
  status = @challenge.status
  return 'invalid' if status == 'invalid'

  'valid'
rescue StandardError => e
  Rails.logger.info "Error validating cerficate #{e.message}"
  'error'
end
```

# What actually went wrong?

Despite the efforts to look into the code, I had no idea what the cause of the problem was. In the meantime, I tried tweaking a few settings from the Cloudflare Dashboard.

Now that I know HTTP-01 challenges are performed, I created some rules for endpoints with the expected location, based on [a forum post](https://forum.hestiacp.com/t/error-let-s-encrypt-finalize-bad-status-403-mail-domain/16274/6 "Error: Let’s Encrypt finalize bad status 403 - mail domain - Community Support - Hestia Control Panel - Discourse") I have come across. I created a configuration rule with the following details:

- Rule name: Disable HTTPS for ACME Challenges
- If incoming requests match…: Custom filter expression
	- Field: URI Path
	- Operator: starts with
	- Value: `/.well-known/acme-challenge/`
	- Expression Preview: `(starts_with(http.request.uri.path, "/.well-known/acme-challenge/"))`
- Then the settings are…
	- Automatic HTTPS Rewrites: off
	- Browser Integrity Check: off
	- Opportunistic Encryption: off
	- SSL: off

A cache rule was also added, with the following details:

- Rule name: Bypass Cache for ACME Challenges
- If incoming requests match…: Custom filter expression
	- Field: URI Path
	- Operator: starts with
	- Value: `/.well-known/acme-challenge/`
	- Expression Preview: `(starts_with(http.request.uri.path, "/.well-known/acme-challenge/"))`
- Then...
	- Cache eligibility: Bypass cache

When the workaround above did not work, I made another request after using "Full (strict)" mode for SSL/TLS encryption and disabling Universal SSL. However, I received the same <s>rejection</s> email saying my domain is not configured correctly.

## Running Certbot

To see if certificates can be issued at all, I decided to try [Certbot](https://certbot.eff.org/ "Certbot"). Before everything, however, I had a quick visit to [Let's Debug](https://letsdebug.net/ "Let's Debug") and entered my domain, only to see it saying "no issues were found" with my domain.

> **All OK!**
> 
> No issues were found with lyuk98.com. If you are having problems with creating an SSL certificate, please visit the [Let's Encrypt Community forums](https://community.letsencrypt.org/) and post a question there.

I started by making a simple web server. Since my home network does not let me forward ports 80 and 443, I went to Google Cloud to [add a new Compute Engine instance](https://console.cloud.google.com/compute/instancesAdd "Overview – Compute Engine – Google Cloud console"). After quickly setting it up and connecting to it via SSH, I installed Apache HTTP Server and Certbot.

```
lyuk98@instance-1:~$ sudo apt install apache2
lyuk98@instance-1:~$ sudo snap install --classic certbot
```

The server was up, and I added the instance's external IP address to my DNS record pointing to an unused subdomain.

![A form at Cloudflare Dashboard for adding a DNS record for my domain. The type is set to A. The name, which is a required field, is set to test. The IPv4 address, which is also a required field, is empty. The proxy status is set to DNS only. TTL is set to Auto.](https://images.lyuk98.com/b74fc8b6-b681-4f44-a7ba-28afaaf1e12b.avif "Adding a new DNS record")

Certbot was then used to issue a certificate using the staging Let's Encrypt server.

```
lyuk98@instance-1:~$ sudo certbot run --apache --test-cert -d test.lyuk98.com
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Enter email address or hit Enter to skip.
 (Enter 'c' to cancel): 

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at:
https://letsencrypt.org/documents/LE-SA-v1.5-February-24-2025.pdf
You must agree in order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: y
Account registered.
Requesting a certificate for test.lyuk98.com

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/test.lyuk98.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/test.lyuk98.com/privkey.pem
This certificate expires on 2025-07-13.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

Deploying certificate
Successfully deployed certificate for test.lyuk98.com to /etc/apache2/sites-available/000-default-le-ssl.conf
Congratulations! You have successfully enabled HTTPS on https://test.lyuk98.com
```

## Self-hosting Listed for debugging

Now that I have ruled out the possibility of being unable to issue certificates at all, it was time to check if something went wrong with Listed. I made another Compute Engine instance and first ensured that Docker was ready there.

```
lyuk98@instance-2:~$ sudo snap install docker
```

The repository was cloned, and the code was ready.

```
lyuk98@instance-2:~$ git clone https://github.com/standardnotes/listed.git
lyuk98@instance-2:~$ cd listed/
```

I followed [the guide](https://github.com/standardnotes/listed/tree/5750e416b82060a6058631a2d7fdfc301c3107f6?tab=readme-ov-file#how-to-run-locally-with-docker "standardnotes/listed at 5750e416b82060a6058631a2d7fdfc301c3107f6") for running the server with Docker. The `.env` was edited to reflect the domain name and the IP address of the self-hosted instance.

```diff
lyuk98@instance-2:~/listed$ nano .env.sample
lyuk98@instance-2:~/listed$ git diff .env.sample
diff --git a/.env.sample b/.env.sample
index 8c02ebc..8db5c21 100644
--- a/.env.sample
+++ b/.env.sample
@@ -15,11 +15,11 @@ DB_ROOT_PASSWORD=changeme123
 
 PORT=3000
 
-HOST=http://localhost:3000
+HOST=http://test.lyuk98.com
 
 # Custom Domains
 LETSENCRYPT_PRIVATE_KEY=
-CUSTOM_DOMAIN_IP=
+CUSTOM_DOMAIN_IP=34.53.34.100
 
 # SSL Certificates Renewal Script
 NGINX_CONFIG_PATH=
lyuk98@instance-2:~/listed$ cp .env.sample .env
```

The server inside the container would be listening to port 3000, but I wanted the host to listen to port 80 for it. I [edited](https://github.com/standardnotes/listed/blob/5750e416b82060a6058631a2d7fdfc301c3107f6/docker-compose.yml "listed/docker-compose.yml at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed") `docker-compose.yml` for appropriate port mapping.

```diff
lyuk98@instance-2:~/listed$ nano docker-compose.yml
lyuk98@instance-2:~/listed$ git diff docker-compose.yml
diff --git a/docker-compose.yml b/docker-compose.yml
index 6ee49b3..33b91c5 100644
--- a/docker-compose.yml
+++ b/docker-compose.yml
@@ -9,7 +9,7 @@ services:
     environment:
       DB_HOST: db
     ports:
-      - ${PORT}:3000
+      - 80:3000
     volumes:
       - .:/listed
   db:
```

It was time to try running the server.

```
lyuk98@instance-2:~/listed$ sudo docker compose up -d
```

<details>
	<summary>The excitement was short-lived, however, as the command failed while building the local image.</summary>
<pre><code>------
 > [app  4/15] RUN apt-get update     && apt-get install -y git build-essential libmariadb-dev curl imagemagick python     && apt-get -y autoclean:
0.420 Ign:1 http://deb.debian.org/debian stretch InRelease
0.422 Ign:2 http://security.debian.org/debian-security stretch/updates InRelease
0.428 Ign:3 http://deb.debian.org/debian stretch-updates InRelease
0.432 Ign:4 http://security.debian.org/debian-security stretch/updates Release
0.438 Ign:5 http://deb.debian.org/debian stretch Release
0.447 Ign:6 http://deb.debian.org/debian stretch-updates Release
0.586 Ign:7 http://security.debian.org/debian-security stretch/updates/main all Packages
0.589 Ign:8 http://deb.debian.org/debian stretch/main amd64 Packages
0.595 Ign:9 http://security.debian.org/debian-security stretch/updates/main amd64 Packages
0.598 Ign:10 http://deb.debian.org/debian stretch/main all Packages
0.740 Ign:11 http://deb.debian.org/debian stretch-updates/main amd64 Packages
0.748 Ign:12 http://deb.debian.org/debian stretch-updates/main all Packages
0.749 Ign:7 http://security.debian.org/debian-security stretch/updates/main all Packages
0.759 Ign:9 http://security.debian.org/debian-security stretch/updates/main amd64 Packages
0.890 Ign:8 http://deb.debian.org/debian stretch/main amd64 Packages
0.898 Ign:10 http://deb.debian.org/debian stretch/main all Packages
0.912 Ign:7 http://security.debian.org/debian-security stretch/updates/main all Packages
0.922 Ign:9 http://security.debian.org/debian-security stretch/updates/main amd64 Packages
1.040 Ign:11 http://deb.debian.org/debian stretch-updates/main amd64 Packages
1.049 Ign:12 http://deb.debian.org/debian stretch-updates/main all Packages
1.076 Ign:7 http://security.debian.org/debian-security stretch/updates/main all Packages
1.086 Ign:9 http://security.debian.org/debian-security stretch/updates/main amd64 Packages
1.191 Ign:8 http://deb.debian.org/debian stretch/main amd64 Packages
1.198 Ign:10 http://deb.debian.org/debian stretch/main all Packages
1.240 Ign:7 http://security.debian.org/debian-security stretch/updates/main all Packages
1.255 Ign:9 http://security.debian.org/debian-security stretch/updates/main amd64 Packages
1.340 Ign:11 http://deb.debian.org/debian stretch-updates/main amd64 Packages
1.349 Ign:12 http://deb.debian.org/debian stretch-updates/main all Packages
1.418 Ign:7 http://security.debian.org/debian-security stretch/updates/main all Packages
1.431 Err:9 http://security.debian.org/debian-security stretch/updates/main amd64 Packages
1.431   404  Not Found [IP: 151.101.66.132 80]
1.496 Ign:8 http://deb.debian.org/debian stretch/main amd64 Packages
1.503 Ign:10 http://deb.debian.org/debian stretch/main all Packages
1.646 Ign:11 http://deb.debian.org/debian stretch-updates/main amd64 Packages
1.654 Ign:12 http://deb.debian.org/debian stretch-updates/main all Packages
1.796 Ign:8 http://deb.debian.org/debian stretch/main amd64 Packages
1.803 Ign:10 http://deb.debian.org/debian stretch/main all Packages
1.945 Ign:11 http://deb.debian.org/debian stretch-updates/main amd64 Packages
1.954 Ign:12 http://deb.debian.org/debian stretch-updates/main all Packages
2.096 Err:8 http://deb.debian.org/debian stretch/main amd64 Packages
2.096   404  Not Found [IP: 151.101.2.132 80]
2.112 Ign:10 http://deb.debian.org/debian stretch/main all Packages
2.260 Err:11 http://deb.debian.org/debian stretch-updates/main amd64 Packages
2.260   404  Not Found [IP: 151.101.2.132 80]
2.268 Ign:12 http://deb.debian.org/debian stretch-updates/main all Packages
2.276 Reading package lists...
2.290 W: The repository 'http://security.debian.org/debian-security stretch/updates Release' does not have a Release file.
2.290 W: The repository 'http://deb.debian.org/debian stretch Release' does not have a Release file.
2.290 W: The repository 'http://deb.debian.org/debian stretch-updates Release' does not have a Release file.
2.290 E: Failed to fetch http://security.debian.org/debian-security/dists/stretch/updates/main/binary-amd64/Packages  404  Not Found [IP: 151.101.66.132 80]
2.290 E: Failed to fetch http://deb.debian.org/debian/dists/stretch/main/binary-amd64/Packages  404  Not Found [IP: 151.101.2.132 80]
2.290 E: Failed to fetch http://deb.debian.org/debian/dists/stretch-updates/main/binary-amd64/Packages  404  Not Found [IP: 151.101.2.132 80]
2.290 E: Some index files failed to download. They have been ignored, or old ones used instead.
------
failed to solve: process "/bin/sh -c apt-get update     && apt-get install -y git build-essential libmariadb-dev curl imagemagick python     && apt-get -y autoclean" did not complete successfully: exit code: 100</code></pre>
</details>

It was weird to see that something as simple as updating the package index failed. I soon found [the solution](https://serverfault.com/questions/1074688/security-debian-org-does-not-have-a-release-file-on-with-debian-docker-images/1130167#1130167 "security.debian.org 'does not have a Release file' on with Debian Docker images - Server Fault"), though, which I applied to `Dockerfile`.

```diff
lyuk98@instance-2:~/listed$ nano Dockerfile
lyuk98@instance-2:~/listed$ git diff Dockerfile
diff --git a/Dockerfile b/Dockerfile
index 5b38d0d..e58ffac 100644
--- a/Dockerfile
+++ b/Dockerfile
@@ -7,6 +7,7 @@ RUN addgroup --system listed --gid $GID && adduser --disabled-password --system
 
 RUN rm /bin/sh && ln -s /bin/bash /bin/sh
 
+RUN echo "deb http://archive.debian.org/debian stretch main contrib non-free" > /etc/apt/sources.list
 RUN apt-get update \
     && apt-get install -y git build-essential libmariadb-dev curl imagemagick python \
     && apt-get -y autoclean
```

<details>
	<summary>Running <code>sudo docker compose up -d</code> again, however, brought me another error.</summary>
	<pre><code>------                                                                                                     
 > [app 13/16] RUN yarn install --pure-lockfile:                                                           
5.249 yarn install v1.22.22                                                                                
5.488 warning package.json: No license field
6.481 warning listed: No license field
6.484 [1/4] Resolving packages...
13.49 [2/4] Fetching packages...
206.5 info There appears to be trouble with your network connection. Retrying...
239.7 info There appears to be trouble with your network connection. Retrying...
265.0 info There appears to be trouble with your network connection. Retrying...
273.0 info There appears to be trouble with your network connection. Retrying...
298.5 info There appears to be trouble with your network connection. Retrying...
306.3 info There appears to be trouble with your network connection. Retrying...
331.7 info There appears to be trouble with your network connection. Retrying...
419.8 info Visit https://yarnpkg.com/en/docs/cli/install for documentation about this command.
420.0 error Error: https://registry.yarnpkg.com/rxjs/-/rxjs-6.6.3.tgz: ESOCKETTIMEDOUT
420.0     at ClientRequest.&lt;anonymous&gt; (/usr/local/nvm/versions/node/v14.18.2/lib/node_modules/yarn/lib/cli.js:142037:19)
420.0     at Object.onceWrapper (events.js:519:28)
420.0     at ClientRequest.emit (events.js:400:28)
420.0     at TLSSocket.emitRequestTimeout (_http_client.js:790:9)
420.0     at Object.onceWrapper (events.js:519:28)
420.0     at TLSSocket.emit (events.js:412:35)
420.0     at TLSSocket.Socket._onTimeout (net.js:495:8)
420.0     at listOnTimeout (internal/timers.js:557:17)
420.0     at processTimers (internal/timers.js:500:7)
------
failed to solve: process "/bin/sh -c yarn install --pure-lockfile" did not complete successfully: exit code: 1</code></pre>
</details>

My Compute Engine instance was apparently too slow that Yarn gave up while trying. Following [the suggestions I found online](https://stackoverflow.com/questions/51508364/yarn-there-appears-to-be-trouble-with-your-network-connection-retrying/51508426#51508426 "yarnpkg - Yarn - There appears to be trouble with your network connection. Retrying - Stack Overflow"), I added `--network-timeout 100000` as an option. Newer versions of Yarn [apparently does this differently](https://yarnpkg.com/configuration/yarnrc#httpTimeout), but I did not bother since it looked like an old one is installed with `npm install -g yarn`.

```diff
lyuk98@instance-2:~/listed$ git add Dockerfile
lyuk98@instance-2:~/listed$ nano Dockerfile
lyuk98@instance-2:~/listed$ git diff Dockerfile
diff --git a/Dockerfile b/Dockerfile
index e58ffac..4eacf74 100644
--- a/Dockerfile
+++ b/Dockerfile
@@ -38,7 +38,7 @@ USER listed
 
 COPY --chown=$UID:$GID package.json yarn.lock Gemfile Gemfile.lock /listed/
 
-RUN yarn install --pure-lockfile
+RUN yarn install --pure-lockfile --network-timeout 100000
 
 RUN gem install bundler && bundle install
 
```

<details>
	<summary>I ran <code>sudo docker compose up -d</code>, but I met yet another error.</summary>
	<pre><code>------                                                                                                     
 > [app 14/16] RUN gem install bundler && bundle install:                                                  
323.5 ERROR:  Error installing bundler:                                                                    
323.5   The last version of bundler (>= 0) to support your Ruby & RubyGems was 2.4.22. Try installing it with `gem install bundler -v 2.4.22`
323.5 	bundler requires Ruby version >= 3.1.0. The current ruby version is 2.6.5.114.
------
failed to solve: process "/bin/sh -c gem install bundler && bundle install" did not complete successfully: exit code: 1</code></pre>
</details>

It looked like this project's dependencies are quite old. I did exactly what the error message suggested.

```diff
lyuk98@instance-2:~/listed$ git add Dockerfile
lyuk98@instance-2:~/listed$ nano Dockerfile
lyuk98@instance-2:~/listed$ git diff Dockerfile
diff --git a/Dockerfile b/Dockerfile
index 4eacf74..8532a4a 100644
--- a/Dockerfile
+++ b/Dockerfile
@@ -40,7 +40,7 @@ COPY --chown=$UID:$GID package.json yarn.lock Gemfile Gemfile.lock /listed/
 
 RUN yarn install --pure-lockfile --network-timeout 100000
 
-RUN gem install bundler && bundle install
+RUN gem install bundler -v 2.4.22 && bundle install
 
 COPY --chown=$UID:$GID . /listed
 
```

<details>
	<summary>
		<code>sudo docker compose up -d</code> finally completed, but the web page was still unreachable. I checked the logs to find out that the container was waiting for a command that could never succeed.
	</summary>
	<pre><code>lyuk98@instance-2:~/listed$ sudo docker logs listed-app-1
./wait-for.sh: line 11: nc: command not found
db:3306 is unavailable yet - waiting for it to start
./wait-for.sh: line 11: nc: command not found
db:3306 is unavailable yet - waiting for it to start
./wait-for.sh: line 11: nc: command not found
db:3306 is unavailable yet - waiting for it to start
./wait-for.sh: line 11: nc: command not found
db:3306 is unavailable yet - waiting for it to start
./wait-for.sh: line 11: nc: command not found
db:3306 is unavailable yet - waiting for it to start</code></pre>
</details>

The service was shut down and the `docker-compose.yml` was once again edited. Instead of relying on a shell script, I used `depends_on` to manage dependency. Since the images were already built, I [forced a rebuild](https://stackoverflow.com/questions/36884991/how-to-rebuild-docker-container-in-docker-compose-yml/50802581#50802581 "How to rebuild docker container in docker-compose.yml? - Stack Overflow") by adding a few options to the `docker compose up` command.

```diff
lyuk98@instance-2:~/listed$ sudo docker compose down
lyuk98@instance-2:~/listed$ git add docker-compose.yml
lyuk98@instance-2:~/listed$ nano docker-compose.yml
lyuk98@instance-2:~/listed$ git diff docker-compose.yml
diff --git a/docker-compose.yml b/docker-compose.yml
index 33b91c5..3976b6b 100644
--- a/docker-compose.yml
+++ b/docker-compose.yml
@@ -3,7 +3,9 @@ services:
   app:
     build:
       context: .
-    entrypoint: ["./wait-for.sh", "db", "3306", "./docker/entrypoint.sh", "start-local"]
+    entrypoint: ["./docker/entrypoint.sh", "start-local"]
+    depends_on:
+      - db
     env_file: .env
     restart: unless-stopped
     environment:
lyuk98@instance-2:~/listed$ sudo docker compose up -d --build --force-recreate -d app
```

<details>
	<summary>The containers started again, but I faced another error where mounted directory could not be written.</summary>
	<pre><code>lyuk98@instance-2:~/listed$ sudo docker logs listed-app-1
Prestart Step 1/5 - Removing server lock
Prestart Step 2/5 - Installing dependencies
yarn install v1.22.22
warning package.json: No license field
warning listed: No license field
[1/4] Resolving packages...
[2/4] Fetching packages...
[3/4] Linking dependencies...
warning " > eslint-plugin-prettier@4.2.1" has incorrect peer dependency "eslint@>=7.28.0".
error Error: EACCES: permission denied, mkdir '/listed/node_modules'
info Visit https://yarnpkg.com/en/docs/cli/install for documentation about this command.
Prestart Step 1/5 - Removing server lock
Prestart Step 2/5 - Installing dependencies
yarn install v1.22.22
warning package.json: No license field
warning listed: No license field
[1/4] Resolving packages...
[2/4] Fetching packages...
[3/4] Linking dependencies...
warning " > eslint-plugin-prettier@4.2.1" has incorrect peer dependency "eslint@>=7.28.0".
error Error: EACCES: permission denied, mkdir '/listed/node_modules'
info Visit https://yarnpkg.com/en/docs/cli/install for documentation about this command.
Prestart Step 1/5 - Removing server lock
Prestart Step 2/5 - Installing dependencies</code></pre>
</details>

I did not really need persistent data for this test anyway, so I removed the `volumes` section altogether.

```diff
lyuk98@instance-2:~/listed$ sudo docker compose down
lyuk98@instance-2:~/listed$ git add docker-compose.yml
lyuk98@instance-2:~/listed$ nano docker-compose.yml
lyuk98@instance-2:~/listed$ git diff docker-compose.yml
diff --git a/docker-compose.yml b/docker-compose.yml
index 3976b6b..47829f7 100644
--- a/docker-compose.yml
+++ b/docker-compose.yml
@@ -12,8 +12,6 @@ services:
       DB_HOST: db
     ports:
       - 80:3000
-    volumes:
-      - .:/listed
   db:
     image: mysql:5.6
     environment:
```

With `sudo docker compose up -d`, the containers were up and the server was finally reachable via web.

It was time to feed some sample data. I first entered the database container, running `mysql` with the default credentials set by [the sample dotenv configuration](https://github.com/standardnotes/listed/blob/5750e416b82060a6058631a2d7fdfc301c3107f6/.env.sample "listed/.env.sample at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed") `.env.sample`.

```
lyuk98@instance-2:~/listed$ sudo docker exec -it listed-db-1 mysql -u std_listed_user --password=changeme123
```

Following [the instruction](https://github.com/standardnotes/listed/tree/5750e416b82060a6058631a2d7fdfc301c3107f6?tab=readme-ov-file#seeding-data "standardnotes/listed at 5750e416b82060a6058631a2d7fdfc301c3107f6") by Listed, I issued some SQL statements to the database.

```
mysql> use sn_listed;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> truncate table subscriptions;
Query OK, 0 rows affected (0.09 sec)

mysql> truncate table subscribers;
Query OK, 0 rows affected (0.02 sec)

mysql> truncate table authors;
Query OK, 0 rows affected (0.03 sec)

mysql> insert into authors (secret, email, email_verified, created_at, updated_at) values ('secret1', 'author1@example.com', true, NOW(), NOW());
Query OK, 1 row affected (0.01 sec)

mysql> select id from authors;
+----+
| id |
+----+
|  1 |
+----+
1 row in set (0.00 sec)

mysql> quit
Bye
```

The new user's settings was now reachable with `http://test.lyuk98.com/authors/1/settings?secret=secret1`.

A simple `A` record for an unused `blog` subdomain was added to test custom-domain registration.

![A form at Cloudflare Dashboard for adding a DNS record for my domain. The type is set to A. The name, which is a required field, is set to blog. The IPv4 address, which is also a required field, is blank. The proxy status is set to DNS only. TTL is set to Auto.](https://images.lyuk98.com/1d8f7cf7-4dea-4e24-8322-c4370b06b843.avif "Adding an A record for the new subdomain")

I filled up and submitted the familiar custom-domain form to see what happens.

![A section of the blog's settings titled Custom domain. There are two blank fields which are labelled "Standard Notes account email address" and "Your domain", respectively. The Submit button is disabled. The description is as follows: "Custom domains are available for Standard Notes members with an active Productivity or Professional plan. Domains include an HTTPS certificate, and require only a simple DNS record on your end. Before submitting this form, please create an A record with your DNS provider with value 34.53.34.100." Under the form, a message is shown, which states the following: "We've received your domain request (blog.lyuk98.com) and will send you an email when your integration is ready (typically up to 1 hour)."](https://images.lyuk98.com/222e56f1-23d8-40fa-a57c-d2cd6152596a.avif "The result after submitting the form")

```
lyuk98@instance-2:~/listed$ sudo docker logs listed-app-1 2> /dev/null | grep SslCertificateCreateJob
I, [2025-04-16T07:04:55.206518 #85]  INFO -- : [ActiveJob] Enqueued SslCertificateCreateJob (Job ID: 650d52b4-8dbd-4e8d-b1fe-0de3ee37fb79) to Async(default) with arguments: "blog.lyuk98.com"
I, [2025-04-16T07:04:55.209398 #85]  INFO -- : [ActiveJob] [SslCertificateCreateJob] [650d52b4-8dbd-4e8d-b1fe-0de3ee37fb79] Performing SslCertificateCreateJob (Job ID: 650d52b4-8dbd-4e8d-b1fe-0de3ee37fb79) from Async(default) with arguments: "blog.lyuk98.com"
D, [2025-04-16T07:04:55.229397 #85] DEBUG -- : [ActiveJob] [SslCertificateCreateJob] [650d52b4-8dbd-4e8d-b1fe-0de3ee37fb79]   [1m[36mSSLCertificate Load (0.6ms)[0m  [1m[34mSELECT  `letsencrypt_certificates`.* FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`domain` = 'blog.lyuk98.com' LIMIT 1[0m
D, [2025-04-16T07:04:55.247752 #85] DEBUG -- : [ActiveJob] [SslCertificateCreateJob] [650d52b4-8dbd-4e8d-b1fe-0de3ee37fb79]   [1m[35m (0.4ms)[0m  [1m[35mBEGIN[0m
D, [2025-04-16T07:04:55.250518 #85] DEBUG -- : [ActiveJob] [SslCertificateCreateJob] [650d52b4-8dbd-4e8d-b1fe-0de3ee37fb79]   [1m[36mLetsEncrypt::Certificate Exists (1.5ms)[0m  [1m[34mSELECT  1 AS one FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`domain` = BINARY 'blog.lyuk98.com' LIMIT 1[0m
D, [2025-04-16T07:04:57.049842 #85] DEBUG -- : [ActiveJob] [SslCertificateCreateJob] [650d52b4-8dbd-4e8d-b1fe-0de3ee37fb79]   [1m[36mSSLCertificate Create (0.6ms)[0m  [1m[32mINSERT INTO `letsencrypt_certificates` (`domain`, `key`, `created_at`, `updated_at`) VALUES ('blog.lyuk98.com', '-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEA5ytxr3OLR2Y5Y8i/sBrG3SVDY9nfdjowfMkwHGhIBISjm8/0\nP156fjIX9Gc+ASW7CAXSnHQo2O2A8UxFEF9hc1YtNHEg0NgESapqlDU3qxhxffDp\nt4FH+LwOLFMxTWylSF9WYwOzJOVS16QCPhH5OPcJGWfkVLeg21XsmtIoCVnfYlR4\ntlC+Xn+yIJShejppYvGk8Eh+aYjTMq1Orst3qKWxPkX/TII72a6namICpKJZ3ZxF\nx4+Q6xbkT8OwsNObfFi6V7IQSXKWgHb0Sn50vJ53xYn6DRr9SBc/v7ff11A27agp\nrP63J1ETx4hjEBJ7S5dSJA1YbrkbICu5P/TX3wIDAQABAoIBAHTgqz8JDT9ROOzx\nf7FbKHaBM5xVeLz+2KsO0WtbciYOpeXOc3BipU4Op7vjQx8zY2fAAecmd8yN8GaP\nqE+J2eyFgp+EHxJYVXqlVfOPIJE574+8cX5dN/VTp1rTyRabOsnnofa31ShvZb4v\nZw7Y6YfaptgYhgIrQYID5He2j5WBy9HmOSnK9Ux/5gKbYkFTNPJI9Wt+gY0JHKa7\ndHphV5nVA/TrGlb4+Gy2LuOQ2aKWBPlOQqjI5+5bsHbEQMFt6L8AAh1GHe1TJerA\n6OrFkQx3T+0bwcAps/mnAIZRbT3sQJvrqyqGCfmco0lKe3IjBAMD3fOdcLgjstnY\nnmZ4dBECgYEA/ZsfaovoA9I+2xns835/7yrKGEDOjaF0sZjUzglH7+iE5YNdSNlR\nYQuOMIZPu1S5vmcpJTdDSxSR3lQKivI/Gf3d8E82Qc5eqCv+geog8y87jrzNzPXH\nRCLC+XvAPw6HZprIdcB4B19yJyObDoJqaNw2egrym9PZa4S1KRqTNSkCgYEA6VoZ\nzDGIEITXsXtKeX7NvHVAj5BRbNs/1uhRS+MdRP0yd3yVSxzL8Vgjit1OMxr5EyCy\nhtrrMMJtZBj33OIfRUdPdgjCjT1BCoHgmmY/q3UrnsbN2LVtgu4QxXSmJUoq3CSj\ntcLfs4+6qszYeOJAmNICS0H00qSHK2thfBa1/ccCgYBtTWxO6ZnH+9enaxcbIwxU\nsmaD6XqcxFedK7ecTZe5qMeOe/26ph9S6j4QX/MBVFTx4Vh0d8sDEwyDfElG9X2I\n4EfFiP5jgmR9quh4acZlyZerv2gbzFpj3W+XQ2TqSILHEDMRvTB+TP7QK6JqsH7Y\nTwETvKAv1TDCDGJgItoLcQKBgQDGKZOayb1IeedJeu/FuR8xmUjYIbBkBtRxxhuz\nnAyxF2uR+KQ3gx7VtwmH1WOhFpjJ24x/5MyxPYrz5Bgo5YW0qVgbXlkI5CmlqKF5\nvLb4/amrThxkmb2D4HMxm1u0cwVuqVa09eZOcBIPFaIHFevRWxZDnqEveDSpdKj2\nXbry5QKBgQDGJm0IvxntL22zQ0Y0nOTKitJwAp2+tC/7+fiUzEUXkx/bbpZhJ0pZ\nh3aVdfoXS/npANYexnXSMtQI0qusjjqmLeFf7r68FHmQ3TomvOklH0kM4YeZdAxf\nJdpMjXJYTLi5rPlcqcDsGmTImVvUKs/QNs7HNf+YBEo9QEPidTJPow==\n-----END RSA PRIVATE KEY-----\n', '2025-04-16 07:04:55', '2025-04-16 07:04:55')[0m
D, [2025-04-16T07:04:57.053499 #85] DEBUG -- : [ActiveJob] [SslCertificateCreateJob] [650d52b4-8dbd-4e8d-b1fe-0de3ee37fb79]   [1m[35m (2.9ms)[0m  [1m[35mCOMMIT[0m
I, [2025-04-16T07:04:57.053903 #85]  INFO -- : [ActiveJob] [SslCertificateCreateJob] [650d52b4-8dbd-4e8d-b1fe-0de3ee37fb79] Performed SslCertificateCreateJob (Job ID: 650d52b4-8dbd-4e8d-b1fe-0de3ee37fb79) from Async(default) in 1844.33ms
```

The `SslCertificateCreateJob` was "performed" but I could not see any log indicating the actual job being done. Since I did not wish to wait until it is done, I slightly changed the code to perform the job as soon as possible.

```diff
lyuk98@instance-2:~/listed$ sudo docker compose down
lyuk98@instance-2:~/listed$ nano app/controllers/authors_controller.rb
lyuk98@instance-2:~/listed$ git diff app/controllers/authors_controller.rb
diff --git a/app/controllers/authors_controller.rb b/app/controllers/authors_controller.rb
index c6b7e8a..bd345a1 100644
--- a/app/controllers/authors_controller.rb
+++ b/app/controllers/authors_controller.rb
@@ -367,7 +367,7 @@ class AuthorsController < ApplicationController
     @author.domain.active = false
     @author.domain.save
 
-    SslCertificateCreateJob.perform_later(params[:domain])
+    SslCertificateCreateJob.perform_now(params[:domain])
 
     redirect_to_authenticated_settings(@author)
   end
lyuk98@instance-2:~/listed$ sudo docker compose up -d --build --force-recreate -d app
```

I could not submit another custom domain request since the domain name was "taken". Tables containing the domain was truncated to give myself another try.

```
lyuk98@instance-2:~/listed$ sudo docker exec -it listed-db-1 mysql -u std_listed_user --password=changeme123
mysql> use sn_listed;
mysql> truncate table letsencrypt_certificates;
mysql> truncate table domains;
```

Another request for the custom domain was submitted, but there was still no significant work being done. I read [an appropriate reference](https://api.rubyonrails.org/classes/ActiveRecord/Relation.html#method-i-find_or_create_by "ActiveRecord::Relation") and edited [`ssl_certificate_create_job.rb`](https://github.com/standardnotes/listed/blob/5750e416b82060a6058631a2d7fdfc301c3107f6/app/jobs/ssl_certificate_create_job.rb "listed/app/jobs/ssl_certificate_create_job.rb at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed") to let the server perform validation right away.

```diff
lyuk98@instance-2:~/listed$ sudo docker compose down
lyuk98@instance-2:~/listed$ nano app/jobs/ssl_certificate_create_job.rb
lyuk98@instance-2:~/listed$ git diff app/jobs/ssl_certificate_create_job.rb
diff --git a/app/jobs/ssl_certificate_create_job.rb b/app/jobs/ssl_certificate_create_job.rb
index 920985e..fee04aa 100644
--- a/app/jobs/ssl_certificate_create_job.rb
+++ b/app/jobs/ssl_certificate_create_job.rb
@@ -1,5 +1,7 @@
 class SslCertificateCreateJob < ApplicationJob
   def perform(domain)
-    SSLCertificate.find_or_create_by(domain: domain)
+    SSLCertificate.find_or_create_by(domain: domain) do |cert|
+      cert.validate
+    end
   end
 end
lyuk98@instance-2:~/listed$ sudo docker compose up -d --build --force-recreate -d app
```

<details>
	<summary>The domain records were truncated again before giving it another try.</summary>
	<pre><code>lyuk98@instance-2:~/listed$ sudo docker exec -it listed-db-1 mysql -u std_listed_user --password=changeme123
mysql> use sn_listed;
mysql> truncate table letsencrypt_certificates;
mysql> truncate table domains;</code></pre>
</details>

Looking at the logs after the form submission, something was finally being done. However, what greeted me was another error message.

```
lyuk98@instance-2:~/listed$ sudo docker logs listed-app-1 2> /dev/null | grep SslCertificateCreateJob | grep Error
I, [2025-04-16T07:20:30.491956 #85]  INFO -- : [ActiveJob] [SslCertificateCreateJob] [08c427ef-ffec-4061-95e1-e87f02501612] Error validating cerficate No account exists with the provided key
```

Right, I need a Let's Encrypt account. I followed [the guide](https://github.com/elct9620/rails-letsencrypt/tree/715ea48650fd2244eab927288dc121c09afe3411?tab=readme-ov-file#installation "elct9620/rails-letsencrypt at 715ea48650fd2244eab927288dc121c09afe3411") to make one.

```
lyuk98@instance-2:~/listed$ sudo docker exec -it listed-app-1 rails generate lets_encrypt:register
Called 'load' without the :safe option -- defaulting to safe mode.
You can avoid this warning in the future by setting the SafeYAML::OPTIONS[:default_mode] option (to :safe or :unsafe).
Running via Spring preloader in process 148
Starting register Let's Encrypt account
Do you want to use in production environment? [y/N]: n
Where you to save private key [/listed/config/letsencrypt.key]: 
Overwrite /listed/config/letsencrypt.key? (enter "h" for help) [Ynaqh] y
I, [2025-04-16T07:29:25.903445 #148]  INFO -- : [LetsEncrypt] Created new private key for Let's Encrypt
Your privated key is saved in /listed/config/letsencrypt.key, make sure setup configure for your rails.
What email you want to register: admin@test.lyuk98.com
I, [2025-04-16T07:29:35.218643 #148]  INFO -- : [LetsEncrypt] Successfully registered private key with address admin@test.lyuk98.com
Register successed, don't forget backup your private key
```

A new private key was made during the process. I copied the key to `.env` and restarted the containers.

```
lyuk98@instance-2:~/listed$ sudo docker exec listed-app-1 cat /listed/config/letsencrypt.key
lyuk98@instance-2:~/listed$ sudo docker compose down
lyuk98@instance-2:~/listed$ nano .env
lyuk98@instance-2:~/listed$ sudo docker compose up -d
```

<details>
	<summary>The domain records were wiped again. It was tedious, but I kept doing so in a hope to finish working on this problem soon.</summary>
	<pre><code>lyuk98@instance-2:~/listed$ sudo docker exec -it listed-db-1 mysql -u std_listed_user --password=changeme123
mysql> use sn_listed;
mysql> truncate table letsencrypt_certificates;
mysql> truncate table domains;</code></pre>
</details>

The server was up. Submitting the form took longer than the previous attempt.

```
lyuk98@instance-2:~/listed$ sudo docker logs listed-app-1 2> /dev/null | grep -i sslcertificate
I, [2025-04-16T07:37:37.396288 #85]  INFO -- : [ActiveJob] [SslCertificateCreateJob] [dda280ab-8b93-4431-a57d-6d19b6e7f4b8] Performing SslCertificateCreateJob (Job ID: dda280ab-8b93-4431-a57d-6d19b6e7f4b8) from Async(default) with arguments: "blog.lyuk98.com"
D, [2025-04-16T07:37:37.407710 #85] DEBUG -- : [ActiveJob] [SslCertificateCreateJob] [dda280ab-8b93-4431-a57d-6d19b6e7f4b8]   [1m[36mSSLCertificate Load (0.7ms)[0m  [1m[34mSELECT  `letsencrypt_certificates`.* FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`domain` = 'blog.lyuk98.com' LIMIT 1[0m
D, [2025-04-16T07:37:38.516229 #85] DEBUG -- : [ActiveJob] [SslCertificateCreateJob] [dda280ab-8b93-4431-a57d-6d19b6e7f4b8]   [1m[35m (0.5ms)[0m  [1m[35mBEGIN[0m
D, [2025-04-16T07:37:38.518690 #85] DEBUG -- : [ActiveJob] [SslCertificateCreateJob] [dda280ab-8b93-4431-a57d-6d19b6e7f4b8]   [1m[36mLetsEncrypt::Certificate Exists (0.7ms)[0m  [1m[34mSELECT  1 AS one FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`domain` = BINARY 'blog.lyuk98.com' LIMIT 1[0m
D, [2025-04-16T07:37:39.579593 #85] DEBUG -- : [ActiveJob] [SslCertificateCreateJob] [dda280ab-8b93-4431-a57d-6d19b6e7f4b8]   [1m[36mSSLCertificate Create (0.7ms)[0m  [1m[32mINSERT INTO `letsencrypt_certificates` (`domain`, `key`, `verification_path`, `verification_string`, `created_at`, `updated_at`) VALUES ('blog.lyuk98.com', '-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAovX7ehClCZnzQCnpuvxPACZ0SWnmpalxn0777x7wwlRKjrn9\njco4fjLTI96GUcafd8/rMw0OhzrZ4wLkWoLnOprRiSjHf3UxcXRe64Q6i1uVdHO+\nzBGq/0+pJXRW5oSMcng3bv56nwj91opWdH8eh36/7CryWytOgcsk4UnDs7gb6Gju\nN7hL9c0XW2uRP57FEw07L71b3nEoZtFqIMbHEqzHU3/nVMViBwJAkqI2qR/eKdfh\nAEkmSgwYsN62scJDI8AH+IstVoGJVpvcFwAGu7Vv8rmnyqUwQ1SBloR/I1cyVM77\nuCOMEPz/f1kZOTmN7T0zTmD48mEWhAiTT0uYBQIDAQABAoIBADd1v8ArKf+6hS6x\nFPquI7TJYYoaoISAxkqRduxKe2WnijhI1CINUGyin3j1ooDyOBNuj30wVGFxhfXc\nZhrnsgof5m/nkP2vxMP39tXwinwjDxoyyhxpZui9E7PLhEevlJzgjP0ZXmIBjWIW\ncpXzLVCvsmGNvC2K74z8tfB2SkQ/OS3Olxh62svo1Me/4MPAKDokqOlyIg2g6aE3\no2eADdmcTlDBxUdR9iFHze9cpGw2YQBytCcX8O8VuwcV5/ySYHbVk46nurAn9WCB\nokheWMLQuJIAh1c7uVo+py6ALRVO1MFA8FzGOtLav2Cb0fNlJmtv1WGJBDlvzVFk\nn6zhcAECgYEA0ZpwoO7SwD/y33ub2lmB3sG29rNAcmWKAKoSgpbrKbQvwSrp/0Du\nql4oOhRFBgCx5q2ZHcx7m3+i1oC8OBSRW/qjFsTwTEOy8X/qVqueEAytXfwsVmR/\ngKe83EBM6p6YralX5h55mpLWZwXDbfD+U995ULG2UbeaUtVdEfbJUOECgYEAxwh2\nl8NdbYPcC/oE36BRSNEBY4s8gSpE6gImDzctpGuYHMfN2UgYH7+lsh5hXtWqautO\nr86o56JM3HxWG/f51/79PHJ3JR1tkMNUkLDK/sAvbEjnYUl2qpS0pS+RP8utgeoF\nweJ+syIUwaofMG7mFjyFzdgvi6ehlzoImcDOV6UCgYEAsV5BZM30JZ93xMny7ujD\nT18ZltXE+YkXKMzCcSOIyHej2ZCZBtlJnX2kCNHSPuwjnxLT+TVqfAGcKGwz2jj9\ncJo9nCz3M3IuYNJf2QvM68PuiRO16T2N768B0FfRPtEKXhppOWAcg0Myj2d/Iu/G\nJ+9512Eq6Se3PdUzttnhLcECgYEAq9P3pm/Igdqbp09S08kxQ58FBu5W7uASHMB8\nIRiu88rbyMUKRvKBuS8YGp01zMzD0oiRJyBQG6G3n4ZMRNshvELsVzou+EDerWKk\n6EFpDuPWTTLnZssogn3dMtrNF/l8MrNaAxfJ8FaU+tknEgY756iaj6p66aNv0wIM\nGMkmmu0CgYA0ewFdv4hevkkhmdxetNcozJNGdruX19NLfkyYbNHD1D41/6AH8Md6\nOaMiVg4KZuRrmuI9Mnihn97+LnlK8rqTymEWNOkQjtt/XkJBZTCO2tuCxTm9anCZ\nvYohJdKuNWoSsG7knZPZfq6PPfnD3QAeJHElcSVxULKqm2rYvh30sw==\n-----END RSA PRIVATE KEY-----\n', '.well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E', 'KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E.BKKy-MKz0GrnxYcHwnd0Qs7GrPzQn6Xbko7AMS88WLo', '2025-04-16 07:37:38', '2025-04-16 07:37:38')[0m
D, [2025-04-16T07:37:39.583781 #85] DEBUG -- : [ActiveJob] [SslCertificateCreateJob] [dda280ab-8b93-4431-a57d-6d19b6e7f4b8]   [1m[35m (3.2ms)[0m  [1m[35mCOMMIT[0m
I, [2025-04-16T07:37:39.584155 #85]  INFO -- : [ActiveJob] [SslCertificateCreateJob] [dda280ab-8b93-4431-a57d-6d19b6e7f4b8] [LetsEncrypt] Attempting verification of blog.lyuk98.com
I, [2025-04-16T07:38:14.333428 #85]  INFO -- : [ActiveJob] [SslCertificateCreateJob] [dda280ab-8b93-4431-a57d-6d19b6e7f4b8] [LetsEncrypt] Status remained at pending for 30 checks
D, [2025-04-16T07:38:14.407016 #85] DEBUG -- : [ActiveJob] [SslCertificateCreateJob] [dda280ab-8b93-4431-a57d-6d19b6e7f4b8]   [1m[35m (72.8ms)[0m  [1m[35mBEGIN[0m
D, [2025-04-16T07:38:14.409190 #85] DEBUG -- :   [1m[36mSSLCertificate Load (2.4ms)[0m  [1m[34mSELECT  `letsencrypt_certificates`.* FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`verification_path` = '.well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E' LIMIT 1[0m
D, [2025-04-16T07:38:14.410409 #85] DEBUG -- :   [1m[36mCACHE SSLCertificate Load (0.0ms)[0m  [1m[34mSELECT  `letsencrypt_certificates`.* FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`verification_path` = '.well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E' LIMIT 1[0m  [["verification_path", ".well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E"], ["LIMIT", 1]]
D, [2025-04-16T07:38:14.414436 #85] DEBUG -- : [ActiveJob] [SslCertificateCreateJob] [dda280ab-8b93-4431-a57d-6d19b6e7f4b8]   [1m[36mLetsEncrypt::Certificate Exists (5.5ms)[0m  [1m[34mSELECT  1 AS one FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`domain` = BINARY 'blog.lyuk98.com' AND `letsencrypt_certificates`.`id` != 1 LIMIT 1[0m
D, [2025-04-16T07:38:14.416828 #85] DEBUG -- : [ActiveJob] [SslCertificateCreateJob] [dda280ab-8b93-4431-a57d-6d19b6e7f4b8]   [1m[35m (0.9ms)[0m  [1m[35mCOMMIT[0m
I, [2025-04-16T07:38:14.418466 #85]  INFO -- : [ActiveJob] [SslCertificateCreateJob] [dda280ab-8b93-4431-a57d-6d19b6e7f4b8] Performed SslCertificateCreateJob (Job ID: dda280ab-8b93-4431-a57d-6d19b6e7f4b8) from Async(default) in 37021.94ms
D, [2025-04-16T07:38:14.582640 #85] DEBUG -- :   [1m[36mSSLCertificate Load (2.9ms)[0m  [1m[34mSELECT  `letsencrypt_certificates`.* FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`verification_path` = '.well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E' LIMIT 1[0m
D, [2025-04-16T07:38:14.584295 #85] DEBUG -- :   [1m[36mCACHE SSLCertificate Load (0.0ms)[0m  [1m[34mSELECT  `letsencrypt_certificates`.* FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`verification_path` = '.well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E' LIMIT 1[0m  [["verification_path", ".well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E"], ["LIMIT", 1]]
D, [2025-04-16T07:38:14.585358 #85] DEBUG -- :   [1m[36mSSLCertificate Load (3.8ms)[0m  [1m[34mSELECT  `letsencrypt_certificates`.* FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`verification_path` = '.well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E' LIMIT 1[0m
D, [2025-04-16T07:38:14.587602 #85] DEBUG -- :   [1m[36mCACHE SSLCertificate Load (0.0ms)[0m  [1m[34mSELECT  `letsencrypt_certificates`.* FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`verification_path` = '.well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E' LIMIT 1[0m  [["verification_path", ".well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E"], ["LIMIT", 1]]
D, [2025-04-16T07:38:14.809901 #85] DEBUG -- :   [1m[36mSSLCertificate Load (5.1ms)[0m  [1m[34mSELECT  `letsencrypt_certificates`.* FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`verification_path` = '.well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E' LIMIT 1[0m
D, [2025-04-16T07:38:14.811107 #85] DEBUG -- :   [1m[36mCACHE SSLCertificate Load (0.0ms)[0m  [1m[34mSELECT  `letsencrypt_certificates`.* FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`verification_path` = '.well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E' LIMIT 1[0m  [["verification_path", ".well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E"], ["LIMIT", 1]]
D, [2025-04-16T07:38:14.815069 #85] DEBUG -- :   [1m[36mSSLCertificate Load (6.8ms)[0m  [1m[34mSELECT  `letsencrypt_certificates`.* FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`verification_path` = '.well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E' LIMIT 1[0m
D, [2025-04-16T07:38:14.817393 #85] DEBUG -- :   [1m[36mCACHE SSLCertificate Load (0.0ms)[0m  [1m[34mSELECT  `letsencrypt_certificates`.* FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`verification_path` = '.well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E' LIMIT 1[0m  [["verification_path", ".well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E"], ["LIMIT", 1]]
```

A lot have happened, for sure. The endpoint for the HTTP-01 challenge was apparently ready, which I confirmed by manually accessing the resource.

```
[lyuk98@framework ~]$ curl http://test.lyuk98.com/.well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E
KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E.BKKy-MKz0GrnxYcHwnd0Qs7GrPzQn6Xbko7AMS88WLo
```

Despite everything, there was still no certificate. I then realised that the validation did not include issuing a certificate. On top of that, I also noticed [how certificates can be validated and renewed](https://github.com/standardnotes/listed/blob/5750e416b82060a6058631a2d7fdfc301c3107f6/lib/tasks/renew_ssl_certificates.rake "listed/lib/tasks/renew_ssl_certificates.rake at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed").

```ruby
namespace :ssl do
  desc 'Renews the letsencrypt certificates,
    re-imports them to AWS Certificate Manager and adds to the Load Balancer HTTPS listener'
  task renew: :environment do
    Rails.logger.tagged('RenewSSL') do
      certificates = LetsEncrypt.certificate_model.all

      Rails.logger.info "Found #{certificates.length} certificates"

      renew_certificates(certificates)

      active_domains = certificates.select(&:active?).map(&:domain)
      host_domain = URI.parse(ENV['ALT_HOST']).host
      active_domains = active_domains.reject { |domain| domain == host_domain }

      create_nginx_domain_config_files(active_domains)

      update_main_nginx_config_file(active_domains)

      restart_nginx

      validate_certificates(certificates)
    end
  end

  # ...
end
```

After feeling stupid for a while, I edited the code again. Changes made to `ssl_certificate_create_job.rb` were reverted to let the process be done separately.

```
lyuk98@instance-2:~/listed$ sudo docker compose down
lyuk98@instance-2:~/listed$ git restore --staged app/jobs/ssl_certificate_create_job.rb
lyuk98@instance-2:~/listed$ git restore app/jobs/ssl_certificate_create_job.rb
lyuk98@instance-2:~/listed$ sudo docker compose up -d --build --force-recreate -d app
```

<details>
	<summary>Before going for another domain registration, the existing records were wiped once again.</summary>
	<pre><code>lyuk98@instance-2:~/listed$ sudo docker exec -it listed-db-1 mysql -u std_listed_user --password=changeme123
mysql> use sn_listed;
mysql> truncate table letsencrypt_certificates;
mysql> truncate table domains;</code></pre>
</details>

The custom-domain form was submitted. I looked at available tasks by using `rails` to see what I need to do.

```
lyuk98@instance-2:~/listed$ sudo docker exec listed-app-1 rails --tasks
rails about                                              # List versions of all Rails frameworks and the environment
rails active_storage:install                             # Copy over the migration needed to the application
rails app:template                                       # Applies the template supplied by LOCATION=(/path/to/template) or URL
rails app:update                                         # Update configs and some other initially generated files (or use just update:configs or update:bin)
rails assets:clean[keep]                                 # Remove old compiled assets
rails assets:clobber                                     # Remove compiled assets
rails assets:environment                                 # Load asset compile environment
rails assets:precompile                                  # Compile all the assets named in config.assets.precompile
rails cache_digests:dependencies                         # Lookup first-level dependencies for TEMPLATE (like messages/show or comments/_comment.html)
rails cache_digests:nested_dependencies                  # Lookup nested dependencies for TEMPLATE (like messages/show or comments/_comment.html)
rails db:create                                          # Creates the database from DATABASE_URL or config/database.yml for the current RAILS_ENV (use db:create:all to create all databases in the config). Without RAILS_ENV or when RAILS_ENV is development, it defaults to creating the development and test databases
rails db:drop                                            # Drops the database from DATABASE_URL or config/database.yml for the current RAILS_ENV (use db:drop:all to drop all databases in the config). Without RAILS_ENV or when RAILS_ENV is development, it defaults to dropping the development and test databases
rails db:environment:set                                 # Set the environment value for the database
rails db:fixtures:load                                   # Loads fixtures into the current environment's database
rails db:migrate                                         # Migrate the database (options: VERSION=x, VERBOSE=false, SCOPE=blog)
rails db:migrate:ignore_concurrent                       # Run db:migrate but ignore ActiveRecord::ConcurrentMigrationError errors
rails db:migrate:status                                  # Display status of migrations
rails db:rollback                                        # Rolls the schema back to the previous version (specify steps w/ STEP=n)
rails db:schema:cache:clear                              # Clears a db/schema_cache.yml file
rails db:schema:cache:dump                               # Creates a db/schema_cache.yml file
rails db:schema:dump                                     # Creates a db/schema.rb file that is portable against any DB supported by Active Record
rails db:schema:load                                     # Loads a schema.rb file into the database
rails db:seed                                            # Loads the seed data from db/seeds.rb
rails db:seed_authors                                    # Seed authors and subscriptions
rails db:setup                                           # Creates the database, loads the schema, and initializes with the seed data (use db:reset to also drop the database first)
rails db:structure:dump                                  # Dumps the database structure to db/structure.sql
rails db:structure:load                                  # Recreates the databases from the structure.sql file
rails db:version                                         # Retrieves the current schema version number
rails dev:cache                                          # Toggle development mode caching on/off
rails haml:erb2haml                                      # Convert html.erb to html.haml each file in app/views
rails initializers                                       # Print out all defined initializers in the order they are invoked by Rails
rails letsencrypt:renew                                  # Renew certificates that already expired or expiring soon
rails log:clear                                          # Truncates all/specified *.log files in log/ to zero bytes (specify which logs with LOGS=test,development)
rails middleware                                         # Prints out your Rack middleware stack
rails notes                                              # Enumerate all annotations (use notes:optimize, :fixme, :todo for focus)
rails notes:custom                                       # Enumerate a custom annotation, specify with ANNOTATION=CUSTOM
rails privacy:notify_subscribers                         # Send privacy policy update emails to all subscribers
rails react_on_rails:assets:clobber                      # Delete assets created with webpack, in the generated assetst directory (/app/assets/webpack)
rails react_on_rails:assets:compile_environment          # Create webpack assets before calling assets:environment
rails react_on_rails:assets:delete_broken_symlinks       # Cleans all broken symlinks for the assets in the public asset dir
rails react_on_rails:assets:symlink_non_digested_assets  # Creates non-digested symlinks for the assets in the public asset dir
rails react_on_rails:assets:webpack                      # Compile assets with webpack
rails react_on_rails:locale                              # Generate i18n javascript files
rails restart                                            # Restart app by touching tmp/restart.txt
rails routes                                             # Print out all defined routes in match order, with names
rails secret                                             # Generate a cryptographically secure secret key (this is typically used to generate a secret for cookie sessions)
rails ssl:renew                                          # Renews the letsencrypt certificates,
rails stats                                              # Report code statistics (KLOCs, etc) from the application or engine
rails test                                               # Runs all tests in test folder except system ones
rails test:db                                            # Run tests quickly, but also reset db
rails test:system                                        # Run system tests only
rails time:zones[country_or_offset]                      # List all time zones, list by two-letter country code (`rails time:zones[US]`), or list by UTC offset (`rails time:zones[-8]`)
rails tmp:clear                                          # Clear cache, socket and screenshot files from tmp/ (narrow w/ tmp:cache:clear, tmp:sockets:clear, tmp:screenshots:clear)
rails tmp:create                                         # Creates tmp directories for cache, sockets, and pids
rails webpacker                                          # Lists all available tasks in Webpacker
rails webpacker:binstubs                                 # Installs Webpacker binstubs in this application
rails webpacker:check_binstubs                           # Verifies that webpack & webpack-dev-server are present
rails webpacker:check_node                               # Verifies if Node.js is installed
rails webpacker:check_yarn                               # Verifies if Yarn is installed
rails webpacker:clean[keep]                              # Remove old compiled webpacks
rails webpacker:clobber                                  # Remove the webpack compiled output directory
rails webpacker:compile                                  # Compile JavaScript packs using webpack for production with digests
rails webpacker:info                                     # Provide information on Webpacker's environment
rails webpacker:install                                  # Install Webpacker in this application
rails webpacker:install:angular                          # Install everything needed for Angular
rails webpacker:install:coffee                           # Install everything needed for Coffee
rails webpacker:install:elm                              # Install everything needed for Elm
rails webpacker:install:erb                              # Install everything needed for Erb
rails webpacker:install:react                            # Install everything needed for React
rails webpacker:install:stimulus                         # Install everything needed for Stimulus
rails webpacker:install:svelte                           # Install everything needed for Svelte
rails webpacker:install:typescript                       # Install everything needed for Typescript
rails webpacker:install:vue                              # Install everything needed for Vue
rails webpacker:verify_install                           # Verifies if Webpacker is installed
rails webpacker:yarn_install                             # Support for older Rails versions
rails yarn:install                                       # Install all JavaScript dependencies as specified via Yarn
```

<details>
	<summary>What I needed was <code>rails ssl:renew</code>. I ran the command to see what happens.</summary>
	<pre><code>lyuk98@instance-2:~/listed$ sudo docker exec listed-app-1 rails ssl:renew
Called 'load' without the :safe option -- defaulting to safe mode.
You can avoid this warning in the future by setting the SafeYAML::OPTIONS[:default_mode] option (to :safe or :unsafe).
D, [2025-04-17T07:10:23.448747 #226] DEBUG -- : [RenewSSL]   [1m[35m (0.5ms)[0m  [1m[35mSET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci,  @@SESSION.sql_mode = CONCAT(CONCAT(@@sql_mode, ',STRICT_ALL_TABLES'), ',NO_AUTO_VALUE_ON_ZERO'),  @@SESSION.sql_auto_is_null = 0, @@SESSION.wait_timeout = 2147483[0m
D, [2025-04-17T07:10:23.453031 #226] DEBUG -- : [RenewSSL]   [1m[36mSSLCertificate Load (0.5ms)[0m  [1m[34mSELECT `letsencrypt_certificates`.* FROM `letsencrypt_certificates`[0m
I, [2025-04-17T07:10:23.476628 #226]  INFO -- : [RenewSSL] Found 1 certificates
W, [2025-04-17T07:10:23.477386 #226]  WARN -- : [RenewSSL] [blog.lyuk98.com] Processing error: Permission denied @ dir_s_mkdir - /blog.lyuk98.com
rails aborted!
URI::InvalidURIError: bad URI(is not URI?): nil
/listed/lib/tasks/renew_ssl_certificates.rake:18:in `block (3 levels) in &lt;top (required)&gt;'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `block in tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:28:in `tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `tagged'
/listed/lib/tasks/renew_ssl_certificates.rake:10:in `block (2 levels) in &lt;top (required)&gt;'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands/rake/rake_command.rb:23:in `block in perform'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands/rake/rake_command.rb:20:in `perform'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/command.rb:48:in `invoke'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands.rb:18:in `&lt;top (required)&gt;'
/listed/bin/rails:9:in `require'
/listed/bin/rails:9:in `&lt;top (required)&gt;'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/rails.rb:28:in `load'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/rails.rb:28:in `call'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/command.rb:7:in `call'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client.rb:30:in `run'
/usr/local/bundle/gems/spring-2.1.1/bin/spring:49:in `&lt;top (required)&gt;'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/binstub.rb:11:in `load'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/binstub.rb:11:in `&lt;top (required)&gt;'
/listed/bin/spring:15:in `&lt;top (required)&gt;'
bin/rails:3:in `load'
bin/rails:3:in `&lt;main&gt;'
<br>
Caused by:
NoMethodError: undefined method `to_str' for nil:NilClass
/listed/lib/tasks/renew_ssl_certificates.rake:18:in `block (3 levels) in &lt;top (required)&gt;'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `block in tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:28:in `tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `tagged'
/listed/lib/tasks/renew_ssl_certificates.rake:10:in `block (2 levels) in &lt;top (required)&gt;'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands/rake/rake_command.rb:23:in `block in perform'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands/rake/rake_command.rb:20:in `perform'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/command.rb:48:in `invoke'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands.rb:18:in `&lt;top (required)&gt;'
/listed/bin/rails:9:in `require'
/listed/bin/rails:9:in `&lt;top (required)&gt;'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/rails.rb:28:in `load'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/rails.rb:28:in `call'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/command.rb:7:in `call'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client.rb:30:in `run'
/usr/local/bundle/gems/spring-2.1.1/bin/spring:49:in `&lt;top (required)&gt;'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/binstub.rb:11:in `load'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/binstub.rb:11:in `&lt;top (required)&gt;'
/listed/bin/spring:15:in `&lt;top (required)&gt;'
bin/rails:3:in `load'
bin/rails:3:in `&lt;main&gt;'
Tasks: TOP =&gt; ssl:renew
(See full trace by running task with --trace)</code></pre>
</details>

Right, I did not set environment variables for renewing certificates. Where would the appropriate directories be, though? I just put random paths.

```
lyuk98@instance-2:~/listed$ sudo docker compose down
lyuk98@instance-2:~/listed$ nano .env
lyuk98@instance-2:~/listed$ grep -A 3 SSL .env
# SSL Certificates Renewal Script
NGINX_CONFIG_PATH=/listed/nginx.conf
DOMAINS_FOLDER_PATH=/listed/certificates
CERTIFICATES_FOLDER_PATH=/listed/certificates
lyuk98@instance-2:~/listed$ sudo docker compose up -d
```

I tried the process again, but it took much longer than just the validation. In fact, it took so long that I gave up waiting after about an hour. Looking at the output, it was apparently stuck while getting a certificate issued.

```
lyuk98@instance-2:~/listed$ sudo docker exec listed-app-1 rails ssl:renew
Called 'load' without the :safe option -- defaulting to safe mode.
You can avoid this warning in the future by setting the SafeYAML::OPTIONS[:default_mode] option (to :safe or :unsafe).
D, [2025-04-17T09:50:56.937346 #162] DEBUG -- : [RenewSSL]   [1m[35m (0.6ms)[0m  [1m[35mSET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci,  @@SESSION.sql_mode = CONCAT(CONCAT(@@sql_mode, ',STRICT_ALL_TABLES'), ',NO_AUTO_VALUE_ON_ZERO'),  @@SESSION.sql_auto_is_null = 0, @@SESSION.wait_timeout = 2147483[0m
D, [2025-04-17T09:50:56.941531 #162] DEBUG -- : [RenewSSL]   [1m[36mSSLCertificate Load (0.5ms)[0m  [1m[34mSELECT `letsencrypt_certificates`.* FROM `letsencrypt_certificates`[0m
I, [2025-04-17T09:50:56.963835 #162]  INFO -- : [RenewSSL] Found 1 certificates
I, [2025-04-17T09:50:56.964122 #162]  INFO -- : [RenewSSL] [blog.lyuk98.com] Renewing certificate
D, [2025-04-17T09:50:58.019494 #162] DEBUG -- : [RenewSSL] [blog.lyuk98.com]   [1m[35m (0.4ms)[0m  [1m[35mBEGIN[0m
D, [2025-04-17T09:50:58.023747 #162] DEBUG -- : [RenewSSL] [blog.lyuk98.com]   [1m[36mLetsEncrypt::Certificate Exists (0.7ms)[0m  [1m[34mSELECT  1 AS one FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`domain` = BINARY 'blog.lyuk98.com' AND `letsencrypt_certificates`.`id` != 1 LIMIT 1[0m
D, [2025-04-17T09:50:58.025884 #162] DEBUG -- : [RenewSSL] [blog.lyuk98.com]   [1m[36mSSLCertificate Update (0.6ms)[0m  [1m[33mUPDATE `letsencrypt_certificates` SET `verification_path` = '.well-known/acme-challenge/KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E', `verification_string` = 'KTZlGkgQZ7b6Anwec26UwnRhKLmVYjAEzKsNnsvNp0E.BKKy-MKz0GrnxYcHwnd0Qs7GrPzQn6Xbko7AMS88WLo', `updated_at` = '2025-04-17 09:50:58' WHERE `letsencrypt_certificates`.`id` = 1[0m
D, [2025-04-17T09:50:58.030349 #162] DEBUG -- : [RenewSSL] [blog.lyuk98.com]   [1m[35m (3.7ms)[0m  [1m[35mCOMMIT[0m
I, [2025-04-17T09:50:58.030568 #162]  INFO -- : [RenewSSL] [blog.lyuk98.com] [LetsEncrypt] Attempting verification of blog.lyuk98.com
I, [2025-04-17T09:50:58.183761 #162]  INFO -- : [RenewSSL] [blog.lyuk98.com] [LetsEncrypt] Getting certificate for blog.lyuk98.com
```

Despite getting stuck during the issuance, it would still mean that the verification was successful. My root domain apparently failed at that stage, so what could be happening? I decided to test it myself.

## Testing against the root domain

It was time to try getting the certificate for the production environment. Setting `RAILS_ENV` and `NODE_ENV` to `production`, however, brought several errors that happened by not setting some undocumented environment variables. Listed *is* open-source, but it is probably not the easiest thing to self-host.

While the server was down, I edited one of the configurations to use the production server for Let's Encrypt.

```diff
lyuk98@instance-2:~/listed$ sudo docker compose down
lyuk98@instance-2:~/listed$ nano config/initializers/letsencrypt.rb
lyuk98@instance-2:~/listed$ git diff config/initializers/letsencrypt.rb
diff --git a/config/initializers/letsencrypt.rb b/config/initializers/letsencrypt.rb
index f820419..ec2120f 100644
--- a/config/initializers/letsencrypt.rb
+++ b/config/initializers/letsencrypt.rb
@@ -1,7 +1,7 @@
 LetsEncrypt.config do |config|
   # Using Let's Encrypt staging server or not
   # Default only `Rails.env.production? == true` will use Let's Encrypt production server.
-  config.use_staging = !Rails.env.production?
+  config.use_staging = false
 
   # Set the private key path
   # Default is locate at config/letsencrypt.key
lyuk98@instance-2:~/listed$ sudo docker compose up -d --build --force-recreate -d app
```

Since accounts between staging and production servers cannot be used interchangeably, I registered a new one for the production environment.

```
lyuk98@instance-2:~/listed$ sudo docker exec -it listed-app-1 rails generate lets_encrypt:register
Called 'load' without the :safe option -- defaulting to safe mode.
You can avoid this warning in the future by setting the SafeYAML::OPTIONS[:default_mode] option (to :safe or :unsafe).
Running via Spring preloader in process 121
Starting register Let's Encrypt account
Do you want to use in production environment? [y/N]: y
Where you to save private key [/listed/config/letsencrypt.key]: 
Overwrite /listed/config/letsencrypt.key? (enter "h" for help) [Ynaqh] y
I, [2025-04-18T01:46:05.441265 #121]  INFO -- : [LetsEncrypt] Created new private key for Let's Encrypt
Your privated key is saved in /listed/config/letsencrypt.key, make sure setup configure for your rails.
What email you want to register: admin@test.lyuk98.com
I, [2025-04-18T01:46:17.358155 #121]  INFO -- : [LetsEncrypt] Successfully registered private key with address admin@test.lyuk98.com
Register successed, don't forget backup your private key
lyuk98@instance-2:~/listed$ sudo docker exec listed-app-1 cat /listed/config/letsencrypt.key > config/letsencrypt.key
lyuk98@instance-2:~/listed$ nano .env
lyuk98@instance-2:~/listed$ sudo docker compose restart
```

<details>
	<summary>The existing records were wiped once again, hoping this would be the last time.</summary>
	<pre><code>lyuk98@instance-2:~/listed$ sudo docker exec -it listed-db-1 mysql -u std_listed_user --password=changeme123
mysql> use sn_listed;
mysql> truncate table letsencrypt_certificates;
mysql> truncate table domains;</code></pre>
</details>

I temporarily changed the `A` record for my root domain, and submitted the form.

![A section of the blog's settings titled Custom domain. There are two blank fields which are labelled "Standard Notes account email address" and "Your domain", respectively. The Submit button is disabled. The description is as follows: "Custom domains are available for Standard Notes members with an active Productivity or Professional plan. Domains include an HTTPS certificate, and require only a simple DNS record on your end. Before submitting this form, please create an A record with your DNS provider with value 34.53.34.100." Under the form, a message is shown, which states the following: "We've received your domain request (lyuk98.com) and will send you an email when your integration is ready (typically up to 1 hour)."](https://images.lyuk98.com/4d045f9a-a4b1-4b2c-87db-0e2dd0ad30c0.avif)

<details>
	<summary>The subsequently-run task for renewing certificates almost completed its job, but somehow failed.</summary>
	<pre><code>lyuk98@instance-2:~/listed$ sudo docker exec listed-app-1 rails ssl:renew
Called 'load' without the :safe option -- defaulting to safe mode.
You can avoid this warning in the future by setting the SafeYAML::OPTIONS[:default_mode] option (to :safe or :unsafe).
D, [2025-04-18T06:43:29.901700 #135] DEBUG -- : [RenewSSL]   [1m[35m (0.5ms)[0m  [1m[35mSET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci,  @@SESSION.sql_mode = CONCAT(CONCAT(@@sql_mode, ',STRICT_ALL_TABLES'), ',NO_AUTO_VALUE_ON_ZERO'),  @@SESSION.sql_auto_is_null = 0, @@SESSION.wait_timeout = 2147483[0m
D, [2025-04-18T06:43:29.905746 #135] DEBUG -- : [RenewSSL]   [1m[36mSSLCertificate Load (0.4ms)[0m  [1m[34mSELECT `letsencrypt_certificates`.* FROM `letsencrypt_certificates`[0m
I, [2025-04-18T06:43:29.924514 #135]  INFO -- : [RenewSSL] Found 1 certificates
I, [2025-04-18T06:43:29.927168 #135]  INFO -- : [RenewSSL] [lyuk98.com] Creating fullchain certificate file
I, [2025-04-18T06:43:29.927424 #135]  INFO -- : [RenewSSL] [lyuk98.com] Creating privkey certificate file
I, [2025-04-18T06:43:29.927770 #135]  INFO -- : [RenewSSL] [lyuk98.com] Renewing certificate
D, [2025-04-18T06:43:31.197496 #135] DEBUG -- : [RenewSSL] [lyuk98.com]   [1m[35m (0.4ms)[0m  [1m[35mBEGIN[0m
D, [2025-04-18T06:43:31.201443 #135] DEBUG -- : [RenewSSL] [lyuk98.com]   [1m[36mLetsEncrypt::Certificate Exists (0.7ms)[0m  [1m[34mSELECT  1 AS one FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`domain` = BINARY 'lyuk98.com' AND `letsencrypt_certificates`.`id` != 1 LIMIT 1[0m
D, [2025-04-18T06:43:31.203315 #135] DEBUG -- : [RenewSSL] [lyuk98.com]   [1m[36mSSLCertificate Update (0.5ms)[0m  [1m[33mUPDATE `letsencrypt_certificates` SET `verification_path` = '.well-known/acme-challenge/Dx-h9mhUYnxeOPt_Gxc8yj6-UKcImvCp8v8eqT9W1k8', `verification_string` = 'Dx-h9mhUYnxeOPt_Gxc8yj6-UKcImvCp8v8eqT9W1k8.Ko0Pqo24NVSk_-tHImduTyVQsnPrzoU9NLpW9XMKYQU', `updated_at` = '2025-04-18 06:43:31' WHERE `letsencrypt_certificates`.`id` = 1[0m
D, [2025-04-18T06:43:31.206773 #135] DEBUG -- : [RenewSSL] [lyuk98.com]   [1m[35m (2.8ms)[0m  [1m[35mCOMMIT[0m
I, [2025-04-18T06:43:31.206974 #135]  INFO -- : [RenewSSL] [lyuk98.com] [LetsEncrypt] Attempting verification of lyuk98.com
I, [2025-04-18T06:43:32.522999 #135]  INFO -- : [RenewSSL] [lyuk98.com] [LetsEncrypt] Getting certificate for lyuk98.com
D, [2025-04-18T06:43:33.576801 #135] DEBUG -- : [RenewSSL] [lyuk98.com]   [1m[35m (0.3ms)[0m  [1m[35mBEGIN[0m
D, [2025-04-18T06:43:33.578606 #135] DEBUG -- : [RenewSSL] [lyuk98.com]   [1m[36mLetsEncrypt::Certificate Exists (0.6ms)[0m  [1m[34mSELECT  1 AS one FROM `letsencrypt_certificates` WHERE `letsencrypt_certificates`.`domain` = BINARY 'lyuk98.com' AND `letsencrypt_certificates`.`id` != 1 LIMIT 1[0m
D, [2025-04-18T06:43:33.580625 #135] DEBUG -- : [RenewSSL] [lyuk98.com]   [1m[36mSSLCertificate Update (0.7ms)[0m  [1m[33mUPDATE `letsencrypt_certificates` SET `certificate` = '-----BEGIN CERTIFICATE-----\nMIIFFTCCA/2gAwIBAgISBi5mpMAKCNOHEb0WaXYUscE+MA0GCSqGSIb3DQEBCwUA\nMDMxCzAJBgNVBAYTAlVTMRYwFAYDVQQKEw1MZXQncyBFbmNyeXB0MQwwCgYDVQQD\nEwNSMTAwHhcNMjUwNDE4MDU0NTAyWhcNMjUwNzE3MDU0NTAxWjAVMRMwEQYDVQQD\nEwpseXVrOTguY29tMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAn1RQ\nRpRqIdEveAEhK3AT5+RLdTLrE1ot8agWUaBsmuh6gEmht1Jui5s9PAr0j3nxAXy2\n1kiQ8asub+YhG8dMgi/FdWelWb8hzLloCeOvfWrKTnYF8sCDYukBE+Ut2MzpqxoL\nKr9ySzq+FJrIHr1CFbwXdZOxXcYTL1iDusneVtF6AjrrrTacBU4Ly9yJAUFlNbsp\ntKq16Ri0mxGcDolxn9UpGPNu666E45Y2Njjr4+Tj+OVwOFe9PYgjDOTfQziZs0b9\nDZJryB8hbc5GWkfuQ0U5LArCOMUVw8by0YdqyDl16sYDVnwE1H6y+74dDND4aKuy\n/NhruiQl0urri4JpgQIDAQABo4ICPzCCAjswDgYDVR0PAQH/BAQDAgWgMB0GA1Ud\nJQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAMBgNVHRMBAf8EAjAAMB0GA1UdDgQW\nBBRDE7qLtbBf1uZi0CQgp6Ga/kdKVTAfBgNVHSMEGDAWgBS7vMNHpeS8qcbDpHIM\nEI2iNeHI6DBXBggrBgEFBQcBAQRLMEkwIgYIKwYBBQUHMAGGFmh0dHA6Ly9yMTAu\nby5sZW5jci5vcmcwIwYIKwYBBQUHMAKGF2h0dHA6Ly9yMTAuaS5sZW5jci5vcmcv\nMBUGA1UdEQQOMAyCCmx5dWs5OC5jb20wEwYDVR0gBAwwCjAIBgZngQwBAgEwLgYD\nVR0fBCcwJTAjoCGgH4YdaHR0cDovL3IxMC5jLmxlbmNyLm9yZy82My5jcmwwggEF\nBgorBgEEAdZ5AgQCBIH2BIHzAPEAdwCi4wrkRe+9rZt+OO1HZ3dT14JbhJTXK14b\nLMS5UKRH5wAAAZZHo2X/AAAEAwBIMEYCIQDAuELkbfVu7rZEIcvJQcqNPODcCCCR\ncNDMMsftO6f6tgIhAPbVDqcgh+T7MElpoCZJYE9jf+TI2KJVzUGYA0GaWDMPAHYA\n3dzKNJXX4RYF55Uy+sef+D0cUN/bADoUEnYKLKy7yCoAAAGWR6NmGgAABAMARzBF\nAiBEcfw6yS4atziYu3MzqjhHkvQ7+EFkue4MGQ0AgkZYgAIhAOhEteNB+7zw/lYK\nmEoV2LGEEzYqX/AmsHx9P1y4pq7KMA0GCSqGSIb3DQEBCwUAA4IBAQAYrDWcRkPg\n1NJWWpUsWEoLDUMULDTtmWHFgKSNK7HKdRu3e6IvKxcv80cWJ8qCSd+r7PNoUnpG\nqbBPrLBCPd+OdUX/RIbe6ps5w2Bsn6JJWNW7TLnx3GfU0Ws52maT36OrnM8XXz9T\nVF8HcxiOpe2xuO0+Hqma/dPMY17/4truYXgDpbaP7nfniM+7G2MZ1Mnr5rMabAGV\nATsvM7AvBEpAahtYcuM87iSkX++G/A6sOkO45gQJME4COw+mOYBKEm7+umHH62Bv\nH196PRnQG3XJZgqWWuaKQysxf3Ue/lBgxb5dqCxu3Kh6mBE7WYGpvEgCPYYdHrZw\nVjAdfhFd8xuz\n-----END CERTIFICATE-----\n', `intermediaries` = '-----BEGIN CERTIFICATE-----\nMIIFBTCCAu2gAwIBAgIQS6hSk/eaL6JzBkuoBI110DANBgkqhkiG9w0BAQsFADBP\nMQswCQYDVQQGEwJVUzEpMCcGA1UEChMgSW50ZXJuZXQgU2VjdXJpdHkgUmVzZWFy\nY2ggR3JvdXAxFTATBgNVBAMTDElTUkcgUm9vdCBYMTAeFw0yNDAzMTMwMDAwMDBa\nFw0yNzAzMTIyMzU5NTlaMDMxCzAJBgNVBAYTAlVTMRYwFAYDVQQKEw1MZXQncyBF\nbmNyeXB0MQwwCgYDVQQDEwNSMTAwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEK\nAoIBAQDPV+XmxFQS7bRH/sknWHZGUCiMHT6I3wWd1bUYKb3dtVq/+vbOo76vACFL\nYlpaPAEvxVgD9on/jhFD68G14BQHlo9vH9fnuoE5CXVlt8KvGFs3Jijno/QHK20a\n/6tYvJWuQP/py1fEtVt/eA0YYbwX51TGu0mRzW4Y0YCF7qZlNrx06rxQTOr8IfM4\nFpOUurDTazgGzRYSespSdcitdrLCnF2YRVxvYXvGLe48E1KGAdlX5jgc3421H5KR\nmudKHMxFqHJV8LDmowfs/acbZp4/SItxhHFYyTr6717yW0QrPHTnj7JHwQdqzZq3\nDZb3EoEmUVQK7GH29/Xi8orIlQ2NAgMBAAGjgfgwgfUwDgYDVR0PAQH/BAQDAgGG\nMB0GA1UdJQQWMBQGCCsGAQUFBwMCBggrBgEFBQcDATASBgNVHRMBAf8ECDAGAQH/\nAgEAMB0GA1UdDgQWBBS7vMNHpeS8qcbDpHIMEI2iNeHI6DAfBgNVHSMEGDAWgBR5\ntFnme7bl5AFzgAiIyBpY9umbbjAyBggrBgEFBQcBAQQmMCQwIgYIKwYBBQUHMAKG\nFmh0dHA6Ly94MS5pLmxlbmNyLm9yZy8wEwYDVR0gBAwwCjAIBgZngQwBAgEwJwYD\nVR0fBCAwHjAcoBqgGIYWaHR0cDovL3gxLmMubGVuY3Iub3JnLzANBgkqhkiG9w0B\nAQsFAAOCAgEAkrHnQTfreZ2B5s3iJeE6IOmQRJWjgVzPw139vaBw1bGWKCIL0vIo\nzwzn1OZDjCQiHcFCktEJr59L9MhwTyAWsVrdAfYf+B9haxQnsHKNY67u4s5Lzzfd\nu6PUzeetUK29v+PsPmI2cJkxp+iN3epi4hKu9ZzUPSwMqtCceb7qPVxEbpYxY1p9\n1n5PJKBLBX9eb9LU6l8zSxPWV7bK3lG4XaMJgnT9x3ies7msFtpKK5bDtotij/l0\nGaKeA97pb5uwD9KgWvaFXMIEt8jVTjLEvwRdvCn294GPDF08U8lAkIv7tghluaQh\n1QnlE4SEN4LOECj8dsIGJXpGUk3aU3KkJz9icKy+aUgA+2cP21uh6NcDIS3XyfaZ\nQjmDQ993ChII8SXWupQZVBiIpcWO4RqZk3lr7Bz5MUCwzDIA359e57SSq5CCkY0N\n4B6Vulk7LktfwrdGNVI5BsC9qqxSwSKgRJeZ9wygIaehbHFHFhcBaMDKpiZlBHyz\nrsnnlFXCb5s8HKn5LsUgGvB24L7sGNZP2CX7dhHov+YhD+jozLW2p9W4959Bz2Ei\nRmqDtmiXLnzqTpXbI+suyCsohKRg6Un0RC47+cpiVwHiXZAW+cn8eiNIjqbVgXLx\nKPpdzvvtTnOPlC7SQZSYmdunr3Bf9b77AiC/ZidstK36dRILKz7OA54=\n-----END CERTIFICATE-----\n', `renew_after` = '2025-06-18 05:45:01', `expires_at` = '2025-07-17 05:45:01', `updated_at` = '2025-04-18 06:43:33' WHERE `letsencrypt_certificates`.`id` = 1[0m
D, [2025-04-18T06:43:33.585187 #135] DEBUG -- : [RenewSSL] [lyuk98.com]   [1m[35m (3.5ms)[0m  [1m[35mCOMMIT[0m
I, [2025-04-18T06:43:33.585532 #135]  INFO -- : [RenewSSL] [lyuk98.com] [LetsEncrypt] Certificate issued (expires on 2025-07-17 05:45:01 UTC, will renew after 2025-06-18 05:45:01 UTC)
I, [2025-04-18T06:43:33.585663 #135]  INFO -- : [RenewSSL] [lyuk98.com] Creating fullchain certificate file
I, [2025-04-18T06:43:33.585950 #135]  INFO -- : [RenewSSL] [lyuk98.com] Creating privkey certificate file
rails aborted!
URI::InvalidURIError: bad URI(is not URI?): nil
/listed/lib/tasks/renew_ssl_certificates.rake:18:in `block (3 levels) in &lt;top (required)&gt;'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `block in tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:28:in `tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `tagged'
/listed/lib/tasks/renew_ssl_certificates.rake:10:in `block (2 levels) in &lt;top (required)&gt;'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands/rake/rake_command.rb:23:in `block in perform'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands/rake/rake_command.rb:20:in `perform'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/command.rb:48:in `invoke'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands.rb:18:in `&lt;top (required)&gt;'
/listed/bin/rails:9:in `require'
/listed/bin/rails:9:in `&lt;top (required)&gt;'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/rails.rb:28:in `load'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/rails.rb:28:in `call'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/command.rb:7:in `call'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client.rb:30:in `run'
/usr/local/bundle/gems/spring-2.1.1/bin/spring:49:in `&lt;top (required)&gt;'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/binstub.rb:11:in `load'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/binstub.rb:11:in `&lt;top (required)&gt;'
/listed/bin/spring:15:in `&lt;top (required)&gt;'
bin/rails:3:in `load'
bin/rails:3:in `&lt;main&gt;'
<br>
Caused by:
NoMethodError: undefined method `to_str' for nil:NilClass
/listed/lib/tasks/renew_ssl_certificates.rake:18:in `block (3 levels) in &lt;top (required)&gt;'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `block in tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:28:in `tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `tagged'
/listed/lib/tasks/renew_ssl_certificates.rake:10:in `block (2 levels) in &lt;top (required)&gt;'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands/rake/rake_command.rb:23:in `block in perform'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands/rake/rake_command.rb:20:in `perform'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/command.rb:48:in `invoke'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands.rb:18:in `&lt;top (required)&gt;'
/listed/bin/rails:9:in `require'
/listed/bin/rails:9:in `&lt;top (required)&gt;'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/rails.rb:28:in `load'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/rails.rb:28:in `call'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/command.rb:7:in `call'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client.rb:30:in `run'
/usr/local/bundle/gems/spring-2.1.1/bin/spring:49:in `&lt;top (required)&gt;'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/binstub.rb:11:in `load'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/binstub.rb:11:in `&lt;top (required)&gt;'
/listed/bin/spring:15:in `&lt;top (required)&gt;'
bin/rails:3:in `load'
bin/rails:3:in `&lt;main&gt;'
Tasks: TOP =&gt; ssl:renew
(See full trace by running task with --trace)</code></pre>
</details>

I followed the stack trace to the [problematic line](https://github.com/standardnotes/listed/blob/5750e416b82060a6058631a2d7fdfc301c3107f6/lib/tasks/renew_ssl_certificates.rake#L18 "listed/lib/tasks/renew_ssl_certificates.rake at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed"), and found an undocumented environment variable being used.

```ruby
host_domain = URI.parse(ENV['ALT_HOST']).host
```

What is the difference between `ALT_HOST` and `HOST`, anyway? I had no idea, but I set it to the same value as `HOST` since I did not think its meaning was significant enough during this testing.

```
lyuk98@instance-2:~/listed$ sudo docker compose down
lyuk98@instance-2:~/listed$ nano .env
lyuk98@instance-2:~/listed$ grep ALT_HOST .env
ALT_HOST=http://test.lyuk98.com
lyuk98@instance-2:~/listed$ sudo docker compose up -d
```

<details>
	<summary>The server was up, and I ran the renewal task once more. It <s>unsurprisingly</s> led to another error.</summary>
	<pre><code>lyuk98@instance-2:~/listed$ sudo docker exec listed-app-1 rails ssl:renew
Called 'load' without the :safe option -- defaulting to safe mode.
You can avoid this warning in the future by setting the SafeYAML::OPTIONS[:default_mode] option (to :safe or :unsafe).
D, [2025-04-18T06:52:02.500740 #80] DEBUG -- : [RenewSSL]   [1m[35m (0.6ms)[0m  [1m[35mSET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci,  @@SESSION.sql_mode = CONCAT(CONCAT(@@sql_mode, ',STRICT_ALL_TABLES'), ',NO_AUTO_VALUE_ON_ZERO'),  @@SESSION.sql_auto_is_null = 0, @@SESSION.wait_timeout = 2147483[0m
D, [2025-04-18T06:52:02.506299 #80] DEBUG -- : [RenewSSL]   [1m[36mSSLCertificate Load (0.6ms)[0m  [1m[34mSELECT `letsencrypt_certificates`.* FROM `letsencrypt_certificates`[0m
I, [2025-04-18T06:52:02.540166 #80]  INFO -- : [RenewSSL] Found 1 certificates
I, [2025-04-18T06:52:02.543416 #80]  INFO -- : [RenewSSL] [lyuk98.com] Creating fullchain certificate file
I, [2025-04-18T06:52:02.544078 #80]  INFO -- : [RenewSSL] [lyuk98.com] Creating privkey certificate file
I, [2025-04-18T06:52:02.544879 #80]  INFO -- : [RenewSSL] [lyuk98.com] Certificate is not renewable before 2025-06-18 05:45:01 UTC
I, [2025-04-18T06:52:02.545521 #80]  INFO -- : [RenewSSL] [lyuk98.com] Creating nginx domain config file
rails aborted!
Errno::EISDIR: Is a directory @ rb_sysopen - /listed/certificates/lyuk98.com
/listed/lib/tasks/renew_ssl_certificates.rake:148:in `initialize'
/listed/lib/tasks/renew_ssl_certificates.rake:148:in `open'
/listed/lib/tasks/renew_ssl_certificates.rake:148:in `block (2 levels) in create_nginx_domain_config_files'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `block in tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:28:in `tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `tagged'
/listed/lib/tasks/renew_ssl_certificates.rake:122:in `block in create_nginx_domain_config_files'
/listed/lib/tasks/renew_ssl_certificates.rake:121:in `each'
/listed/lib/tasks/renew_ssl_certificates.rake:121:in `create_nginx_domain_config_files'
/listed/lib/tasks/renew_ssl_certificates.rake:21:in `block (3 levels) in &lt;top (required)&gt;'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `block in tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:28:in `tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `tagged'
/listed/lib/tasks/renew_ssl_certificates.rake:10:in `block (2 levels) in &lt;top (required)&gt;'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands/rake/rake_command.rb:23:in `block in perform'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands/rake/rake_command.rb:20:in `perform'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/command.rb:48:in `invoke'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands.rb:18:in `&lt;top (required)&gt;'
/listed/bin/rails:9:in `require'
/listed/bin/rails:9:in `&lt;top (required)&gt;'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/rails.rb:28:in `load'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/rails.rb:28:in `call'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/command.rb:7:in `call'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client.rb:30:in `run'
/usr/local/bundle/gems/spring-2.1.1/bin/spring:49:in `&lt;top (required)&gt;'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/binstub.rb:11:in `load'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/binstub.rb:11:in `&lt;top (required)&gt;'
/listed/bin/spring:15:in `&lt;top (required)&gt;'
bin/rails:3:in `load'
bin/rails:3:in `&lt;main&gt;'
Tasks: TOP =&gt; ssl:renew
(See full trace by running task with --trace)</code></pre>
</details>

`$DOMAINS_FOLDER_PATH/[domain]` is expected to be a file, when `$CERTIFICATES_FOLDER_PATH/[domain]` was already created as a directory. I should have had a deeper look before assigning them the same value, but I will make an excuse that they were not documented. As a result, `DOMAINS_FOLDER_PATH` was changed to something else.

```
lyuk98@instance-2:~/listed$ sudo docker compose down
lyuk98@instance-2:~/listed$ mkdir domains
lyuk98@instance-2:~/listed$ nano .env
lyuk98@instance-2:~/listed$ grep DOMAINS_FOLDER_PATH .env
DOMAINS_FOLDER_PATH=/listed/domains
lyuk98@instance-2:~/listed$ sudo docker compose up -d --build --force-recreate -d app
```

<details>
	<summary>Renewal was re-attempted, but another error showed up.</summary>
	<pre><code>lyuk98@instance-2:~/listed$ sudo docker exec listed-app-1 rails ssl:renew
Called 'load' without the :safe option -- defaulting to safe mode.
You can avoid this warning in the future by setting the SafeYAML::OPTIONS[:default_mode] option (to :safe or :unsafe).
D, [2025-04-18T15:08:14.187508 #134] DEBUG -- : [RenewSSL]   [1m[35m (0.6ms)[0m  [1m[35mSET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci,  @@SESSION.sql_mode = CONCAT(CONCAT(@@sql_mode, ',STRICT_ALL_TABLES'), ',NO_AUTO_VALUE_ON_ZERO'),  @@SESSION.sql_auto_is_null = 0, @@SESSION.wait_timeout = 2147483[0m
D, [2025-04-18T15:08:14.191766 #134] DEBUG -- : [RenewSSL]   [1m[36mSSLCertificate Load (0.5ms)[0m  [1m[34mSELECT `letsencrypt_certificates`.* FROM `letsencrypt_certificates`[0m
I, [2025-04-18T15:08:14.211260 #134]  INFO -- : [RenewSSL] Found 1 certificates
I, [2025-04-18T15:08:14.214943 #134]  INFO -- : [RenewSSL] [lyuk98.com] Creating fullchain certificate file
I, [2025-04-18T15:08:14.215305 #134]  INFO -- : [RenewSSL] [lyuk98.com] Creating privkey certificate file
I, [2025-04-18T15:08:14.216003 #134]  INFO -- : [RenewSSL] [lyuk98.com] Certificate is not renewable before 2025-06-18 05:45:01 UTC
I, [2025-04-18T15:08:14.218255 #134]  INFO -- : [RenewSSL] [lyuk98.com] Creating nginx domain config file
rails aborted!
Errno::ENOENT: No such file or directory @ rb_sysopen - /listed/nginx.conf
/listed/lib/tasks/renew_ssl_certificates.rake:97:in `read'
/listed/lib/tasks/renew_ssl_certificates.rake:97:in `update_main_nginx_config_file'
/listed/lib/tasks/renew_ssl_certificates.rake:23:in `block (3 levels) in &lt;top (required)&gt;'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `block in tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:28:in `tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `tagged'
/listed/lib/tasks/renew_ssl_certificates.rake:10:in `block (2 levels) in &lt;top (required)&gt;'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands/rake/rake_command.rb:23:in `block in perform'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands/rake/rake_command.rb:20:in `perform'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/command.rb:48:in `invoke'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands.rb:18:in `&lt;top (required)&gt;'
/listed/bin/rails:9:in `require'
/listed/bin/rails:9:in `&lt;top (required)&gt;'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/rails.rb:28:in `load'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/rails.rb:28:in `call'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/command.rb:7:in `call'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client.rb:30:in `run'
/usr/local/bundle/gems/spring-2.1.1/bin/spring:49:in `&lt;top (required)&gt;'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/binstub.rb:11:in `load'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/binstub.rb:11:in `&lt;top (required)&gt;'
/listed/bin/spring:15:in `&lt;top (required)&gt;'
bin/rails:3:in `load'
bin/rails:3:in `&lt;main&gt;'
Tasks: TOP =&gt; ssl:renew
(See full trace by running task with --trace)</code></pre>
</details>

The Nginx configuration file apparently needs to exist before the procedure. [A line](https://github.com/standardnotes/listed/blob/5750e416b82060a6058631a2d7fdfc301c3107f6/lib/tasks/renew_ssl_certificates.rake#L105 "listed/lib/tasks/renew_ssl_certificates.rake at 5750e416b82060a6058631a2d7fdfc301c3107f6 · standardnotes/listed") also suggested a certain requirement, which I tried to follow.

```
lyuk98@instance-2:~/listed$ sudo docker compose down
lyuk98@instance-2:~/listed$ nano nginx.conf
lyuk98@instance-2:~/listed$ cat nginx.conf
#BEGIN DOMAIN INCLUDES#

#END DOMAIN INCLUDES#
lyuk98@instance-2:~/listed$ sudo docker compose up -d --build --force-recreate -d app
```

<details>
	<summary>The re-attempted renewal process failed again, but something was different this time.</summary>
	<pre><code>lyuk98@instance-2:~/listed$ sudo docker exec listed-app-1 rails ssl:renew
Called 'load' without the :safe option -- defaulting to safe mode.
You can avoid this warning in the future by setting the SafeYAML::OPTIONS[:default_mode] option (to :safe or :unsafe).
D, [2025-04-18T15:27:01.643829 #128] DEBUG -- : [RenewSSL]   [1m[35m (0.5ms)[0m  [1m[35mSET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci,  @@SESSION.sql_mode = CONCAT(CONCAT(@@sql_mode, ',STRICT_ALL_TABLES'), ',NO_AUTO_VALUE_ON_ZERO'),  @@SESSION.sql_auto_is_null = 0, @@SESSION.wait_timeout = 2147483[0m
D, [2025-04-18T15:27:01.647945 #128] DEBUG -- : [RenewSSL]   [1m[36mSSLCertificate Load (0.4ms)[0m  [1m[34mSELECT `letsencrypt_certificates`.* FROM `letsencrypt_certificates`[0m
I, [2025-04-18T15:27:01.667501 #128]  INFO -- : [RenewSSL] Found 1 certificates
I, [2025-04-18T15:27:01.670698 #128]  INFO -- : [RenewSSL] [lyuk98.com] Creating fullchain certificate file
I, [2025-04-18T15:27:01.671230 #128]  INFO -- : [RenewSSL] [lyuk98.com] Creating privkey certificate file
I, [2025-04-18T15:27:01.671814 #128]  INFO -- : [RenewSSL] [lyuk98.com] Certificate is not renewable before 2025-06-18 05:45:01 UTC
I, [2025-04-18T15:27:01.674520 #128]  INFO -- : [RenewSSL] [lyuk98.com] Creating nginx domain config file
I, [2025-04-18T15:27:01.709925 #128]  INFO -- : [RenewSSL] Restarting nginx...
sh: sudo: command not found
I, [2025-04-18T15:27:01.722396 #128]  INFO -- : [RenewSSL] Restarted nginx
D, [2025-04-18T15:27:01.735970 #128] DEBUG -- : [RenewSSL] [lyuk98.com]   [1m[36mDomain Load (0.5ms)[0m  [1m[34mSELECT  `domains`.* FROM `domains` WHERE `domains`.`domain` = 'lyuk98.com' LIMIT 1[0m
D, [2025-04-18T15:27:01.796702 #128] DEBUG -- : [RenewSSL] [lyuk98.com]   [1m[36mAuthor Load (0.9ms)[0m  [1m[34mSELECT  `authors`.* FROM `authors` WHERE `authors`.`id` = 1 LIMIT 1[0m
D, [2025-04-18T15:27:01.820447 #128] DEBUG -- : [RenewSSL] [lyuk98.com]   [1m[35m (0.4ms)[0m  [1m[35mBEGIN[0m
D, [2025-04-18T15:27:01.823086 #128] DEBUG -- : [RenewSSL] [lyuk98.com]   [1m[36mDomain Update (1.0ms)[0m  [1m[33mUPDATE `domains` SET `approved` = TRUE, `active` = TRUE, `updated_at` = '2025-04-18 15:27:01' WHERE `domains`.`id` = 1[0m
D, [2025-04-18T15:27:01.827286 #128] DEBUG -- : [RenewSSL] [lyuk98.com]   [1m[35m (3.4ms)[0m  [1m[35mCOMMIT[0m
I, [2025-04-18T15:27:02.028569 #128]  INFO -- : [RenewSSL] [lyuk98.com] [Webpacker] Everything's up-to-date. Nothing to do
I, [2025-04-18T15:27:02.042826 #128]  INFO -- : [RenewSSL] [lyuk98.com] [Webpacker] Everything's up-to-date. Nothing to do
I, [2025-04-18T15:27:02.044716 #128]  INFO -- : [RenewSSL] [lyuk98.com] ZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZ
[react_on_rails] Evaluating code to server render.
JavaScript code used: tmp/server-generated-1.js
ZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZ
<br>
I, [2025-04-18T15:27:02.058230 #128]  INFO -- : [RenewSSL] [lyuk98.com] [Webpacker] Everything's up-to-date. Nothing to do
I, [2025-04-18T15:27:02.072899 #128]  INFO -- : [RenewSSL] [lyuk98.com] [Webpacker] Everything's up-to-date. Nothing to do
I, [2025-04-18T15:27:02.116825 #128]  INFO -- : [RenewSSL] [lyuk98.com] [Webpacker] Everything's up-to-date. Nothing to do
I, [2025-04-18T15:27:02.118235 #128]  INFO -- : [RenewSSL] [lyuk98.com] [react_on_rails] Created JavaScript context with file /listed/public/packs/development/server-bundle.js
I, [2025-04-18T15:27:02.221197 #128]  INFO -- : [RenewSSL] [lyuk98.com] [react_on_rails] RENDERED AuthorsMailerDomainApproved to dom node with id: AuthorsMailerDomainApproved-react-component-472dc275-a3e2-4d23-9d83-a3a82a6ca89f
D, [2025-04-18T15:27:02.423696 #128] DEBUG -- : [RenewSSL] [lyuk98.com] AuthorsMailer#domain_approved: processed outbound mail in 422.1ms
I, [2025-04-18T15:27:02.439582 #128]  INFO -- : [RenewSSL] [lyuk98.com] Sent mail to author1@example.com (15.6ms)
D, [2025-04-18T15:27:02.439674 #128] DEBUG -- : [RenewSSL] [lyuk98.com] Date: Fri, 18 Apr 2025 15:27:02 +0000
From: Listed &lt;mail@listed.to&gt;
Reply-To: help@standardnotes.com
To: author1@example.com
Message-ID: &lt;68026f4669bf6_803280a66fb964579c9@ace09ea50fdc.mail&gt;
Subject: Your custom domain is live!
Mime-Version: 1.0
Content-Type: text/html;
 charset=UTF-8
Content-Transfer-Encoding: quoted-printable
<br>
&lt;!DOCTYPE html&gt;=0D
&lt;html&gt;=0D
  &lt;head&gt;=0D
    &lt;meta http-equiv=3D"Content-Type" content=3D"text/html; charset=3Dutf=
-8" /&gt;=0D
    &lt;style&gt;=0D
=0D
    * {=0D
      box-sizing: border-box;=0D
    }=0D
=0D
    html, body {=0D
     margin: 0;=0D
     font-family: sans-serif;=0D
     line-height: 1.5;=0D
     color: black;=0D
=0D
     font-size: 18px;=0D
     height: 100%;=0D
     padding-bottom: 10px;=0D
     margin-bottom: 10px;=0D
    }=0D
=0D
    body {=0D
      padding: 1rem 2rem 4px 2rem;=0D
      margin-left: auto;=0D
      margin-right: auto;=0D
    }=0D
=0D
    @media screen and (min-width: 42em) and (max-width: 64em) {=0D
      body {=0D
        padding: 2rem 2rem;=0D
        font-size: 17px;=0D
      }=0D
    }=0D
=0D
    @media screen and (max-width: 42em) {=0D
      body {=0D
        padding: 0px 1rem;=0D
        font-size: 17px;=0D
      }=0D
    }=0D
=0D
    .block {=0D
      display: block;=0D
    }=0D
=0D
    h1, h2, h3 {=0D
      margin-bottom: 4px;=0D
      -webkit-margin-after: 4px;=0D
    }=0D
=0D
    .post-content .post-body h1 {=0D
      font-size: 1.2rem;=0D
    }=0D
=0D
    .post-content .post-body h2 {=0D
      font-size: 1.1rem;=0D
    }=0D
=0D
    .post-content .post-body h3 {=0D
      font-size: 1.0rem;=0D
    }=0D
=0D
    .post-content .post-body h4 {=0D
      font-size: 0.9rem;=0D
    }=0D
=0D
    blockquote {=0D
       border-left: 2px solid #cdcdcd;=0D
       padding-left: 20px;=0D
       opacity: 0.8;=0D
       margin-left: 0;=0D
    }=0D
=0D
    h2 a {=0D
      color: black;=0D
    }=0D
=0D
    .bottom-margin-space {=0D
      height: 12px;=0D
    }=0D
=0D
    a.unstyled {=0D
      color: black;=0D
      text-decoration: none;=0D
    }=0D
=0D
    .post-footer {=0D
      font-size: 16px;=0D
    }=0D
=0D
    .links-footer {=0D
      font-size: 16px;=0D
      margin-bottom: 20px;=0D
    }=0D
=0D
    .links-footer a {=0D
      margin-right: 6px;=0D
    }=0D
=0D
    .reaction-links {=0D
      margin-bottom: 14px;=0D
    }=0D
=0D
    .reaction-links .reaction-link {=0D
      margin-right: 10px;=0D
      text-decoration: none;=0D
    }=0D
=0D
    .post {=0D
      margin-bottom: 40px;=0D
    }=0D
=0D
    .post-content {=0D
      margin-top: 20px;=0D
      clear: both;=0D
      line-height: 1.6;=0D
      text-align: left;=0D
      font-weight: normal;=0D
      color: black;=0D
      position: relative;=0D
      overflow: visible;=0D
    }=0D
=0D
    pre {=0D
      font-family: Consolas,monaco,"Ubuntu Mono",courier,monospace!import=
ant;=0D
      padding: 16px;=0D
      overflow: auto;=0D
      font-size: 90%;=0D
      line-height: 1.45;=0D
      background-color: #f7f7f7;=0D
      border-radius: 3px;=0D
    }=0D
=0D
    p code {=0D
      font-family: Consolas,monaco,"Ubuntu Mono",courier,monospace!import=
ant;=0D
      font-size: .75rem;=0D
      line-height: .75rem;=0D
      color: #c25;=0D
      padding: 4px 8px;=0D
      background-color: #f7f7f9;=0D
      border: 1px solid #e1e1e8;=0D
      border-radius: 3px;=0D
    }=0D
=0D
    img {=0D
      display: block;=0D
      margin-top: 30px;=0D
      max-width: 100%;=0D
    }=0D
=0D
    hr {=0D
      display: block;=0D
      border: 0;=0D
      text-align: center;=0D
      box-sizing: content-box;=0D
      border: 0 !important;=0D
    }=0D
=0D
    hr.left {=0D
      text-align: left;=0D
    }=0D
=0D
    hr:before {=0D
      font-family: Georgia,Cambria,"Times New Roman",Times,serif;=0D
      font-weight: 400;=0D
      font-style: italic;=0D
      font-size: 28px;=0D
      letter-spacing: .3em;=0D
      content: '...';=0D
      display: inline-block;=0D
      margin-right: .6em;=0D
      color: black;=0D
      position: relative;=0D
    }=0D
=0D
    &lt;/style&gt;=0D
  &lt;/head&gt;=0D
=0D
  &lt;body&gt;=0D
    &lt;script type=3D"application/json" id=3D"js-react-on-rails-context"&gt;{"=
railsEnv":"development","inMailer":true,"i18nLocale":"en","i18nDefaultLoc=
ale":"en","rorVersion":"11.3.0","rorPro":false,"serverSide":false}&lt;/scrip=
t&gt;=0D
&lt;div id=3D"AuthorsMailerDomainApproved-react-component-472dc275-a3e2-4d23=
-9d83-a3a82a6ca89f"&gt;&lt;div data-reactroot=3D""&gt;&lt;h3&gt;Congratulations!&lt;!-- --&gt;=
 &lt;span role=3D"img" aria-label=3D"party-popper"&gt;=F0=9F=8E=89&lt;/span&gt;&lt;/h3&gt;&lt;=
p&gt;Your custom domain has been approved, and your blog is now live at&lt;!-- =
--&gt; &lt;a href=3D"https://lyuk98.com"&gt;https://lyuk98.com&lt;/a&gt;.&lt;/p&gt;&lt;p&gt;Any ques=
tions? Please feel free to reply directly to this email.&lt;/p&gt;&lt;/div&gt;&lt;/div&gt;=0D=
<br>
      &lt;script type=3D"application/json" class=3D"js-react-on-rails-compon=
ent" data-component-name=3D"AuthorsMailerDomainApproved" data-trace=3D"tr=
ue" data-dom-id=3D"AuthorsMailerDomainApproved-react-component-472dc275-a=
3e2-4d23-9d83-a3a82a6ca89f"&gt;{"url":"https://lyuk98.com"}&lt;/script&gt;=0D
      =0D
&lt;script id=3D"consoleReplayLog"&gt;=0D
console.log.apply(console, ["[SERVER] RENDERED AuthorsMailerDomainApprove=
d to dom node with id: AuthorsMailerDomainApproved-react-component-472dc2=
75-a3e2-4d23-9d83-a3a82a6ca89f"]);=0D
&lt;/script&gt;=0D
=0D
=0D
  &lt;/body&gt;=0D
&lt;/html&gt;=0D
<br>
rails aborted!
Errno::ECONNREFUSED: Connection refused - connect(2) for nil port 25
/usr/local/bundle/gems/mail-2.7.1/lib/mail/network/delivery_methods/smtp.rb:109:in `start_smtp_session'
/usr/local/bundle/gems/mail-2.7.1/lib/mail/network/delivery_methods/smtp.rb:100:in `deliver!'
/usr/local/bundle/gems/mail-2.7.1/lib/mail/message.rb:2159:in `do_delivery'
/usr/local/bundle/gems/mail-2.7.1/lib/mail/message.rb:260:in `block in deliver'
/usr/local/bundle/gems/actionmailer-5.2.5/lib/action_mailer/base.rb:560:in `block in deliver_mail'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/notifications.rb:168:in `block in instrument'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/notifications/instrumenter.rb:23:in `instrument'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/notifications.rb:168:in `instrument'
/usr/local/bundle/gems/actionmailer-5.2.5/lib/action_mailer/base.rb:558:in `deliver_mail'
/usr/local/bundle/gems/mail-2.7.1/lib/mail/message.rb:260:in `deliver'
/usr/local/bundle/gems/actionmailer-5.2.5/lib/action_mailer/message_delivery.rb:114:in `block in deliver_now'
/usr/local/bundle/gems/actionmailer-5.2.5/lib/action_mailer/rescuable.rb:17:in `handle_exceptions'
/usr/local/bundle/gems/actionmailer-5.2.5/lib/action_mailer/message_delivery.rb:113:in `deliver_now'
/listed/app/models/author.rb:216:in `notify_domain'
/listed/lib/tasks/renew_ssl_certificates.rake:88:in `block (2 levels) in validate_certificates'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `block in tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:28:in `tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `tagged'
/listed/lib/tasks/renew_ssl_certificates.rake:66:in `block in validate_certificates'
/usr/local/bundle/gems/activerecord-5.2.5/lib/active_record/relation/delegation.rb:71:in `each'
/usr/local/bundle/gems/activerecord-5.2.5/lib/active_record/relation/delegation.rb:71:in `each'
/listed/lib/tasks/renew_ssl_certificates.rake:65:in `validate_certificates'
/listed/lib/tasks/renew_ssl_certificates.rake:27:in `block (3 levels) in &lt;top (required)&gt;'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `block in tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:28:in `tagged'
/usr/local/bundle/gems/activesupport-5.2.5/lib/active_support/tagged_logging.rb:71:in `tagged'
/listed/lib/tasks/renew_ssl_certificates.rake:10:in `block (2 levels) in &lt;top (required)&gt;'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands/rake/rake_command.rb:23:in `block in perform'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands/rake/rake_command.rb:20:in `perform'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/command.rb:48:in `invoke'
/usr/local/bundle/gems/railties-5.2.5/lib/rails/commands.rb:18:in `&lt;top (required)&gt;'
/listed/bin/rails:9:in `require'
/listed/bin/rails:9:in `&lt;top (required)&gt;'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/rails.rb:28:in `load'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/rails.rb:28:in `call'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client/command.rb:7:in `call'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/client.rb:30:in `run'
/usr/local/bundle/gems/spring-2.1.1/bin/spring:49:in `&lt;top (required)&gt;'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/binstub.rb:11:in `load'
/usr/local/bundle/gems/spring-2.1.1/lib/spring/binstub.rb:11:in `&lt;top (required)&gt;'
/listed/bin/spring:15:in `&lt;top (required)&gt;'
bin/rails:3:in `load'
bin/rails:3:in `&lt;main&gt;'
Tasks: TOP =&gt; ssl:renew
(See full trace by running task with --trace)</code></pre>
</details>

It only failed to send the mail after everything else was done; the certificate itself was successfully issued. While it could have made me happy under a different situation, I was getting lost instead. Why did it even work?

I was not sure what else to look for, so I went to sleep. Before the rest, though, I reverted my root domain's `A` record and submitted a custom-domain request.

---

# The unsatisfying resolution

I woke up and found an email that arrived early in the morning:

> **Congratulations! 🎉**
> 
> Your custom domain has been approved, and your blog is now live at [https://lyuk98.com](https://lyuk98.com).
> 
> Any questions? Please feel free to reply directly to this email.

While it was the end of a problem that have bothered me for over a year, not knowing exactly why the integration worked this time was not very satisfying. Nevertheless, I was glad that this problem was done with.

Will I ever come back to this problem to figure out exactly what went wrong? Probably, if renewing a certificate for my domain ever fails. However, that is still a while away, so I will be focusing on other things for now.
