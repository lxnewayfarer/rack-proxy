A request/response rewriting HTTP proxy. A Rack app. Subclass `Rack::Proxy` and provide your `rewrite_env` and `rewrite_response` methods.

Installation
----

Add the following to your `Gemfile`:

```
gem 'rack-proxy', '~> 0.7.4'
```

Or install:

```
gem install rack-proxy
```

Use Cases
----

Below are some examples of real world use cases for Rack-Proxy. If you have done something interesting, add it to the list below and send a PR.

* Allowing one app to act as central trust authority
  * handle accepting self-sign certificates for internal apps
  * authentication / authorization prior to proxying requests to a blindly trusting backend
  * avoiding CORs complications by proxying from same domain to another backend
* subdomain based pass-through to multiple apps
* Complex redirect rules
   * redirect pages with different extensions (ex: `.php`) to another app
   * useful for handling awkward redirection rules for moved pages
* fan Parallel Requests: turning a single API request to [multiple concurrent backend requests](https://github.com/typhoeus/typhoeus#making-parallel-requests) & merging results.
* inserting or stripping headers required or problematic for certain clients

Options
----

Options can be set when initializing the middleware or overriding a method.


* `:streaming` - defaults to `true`, but does not work on all Ruby versions, recommend to set to `false`
* `:ssl_verify_none` - tell `Net::HTTP` to not validate certs
* `:ssl_version` - tell `Net::HTTP` to set a specific `ssl_version`
* `:backend` - the URI parseable format of host and port of the target proxy backend. If not set it will assume the backend target is the same as the source.
* `:read_timeout` - set proxy timeout it defaults to 60 seconds

To pass in options, when you configure your middleware you can pass them in as an optional hash.

```ruby
Rails.application.config.middleware.use ExampleServiceProxy, backend: 'http://guides.rubyonrails.org', streaming: false
```

Examples
----

See and run the examples below from `lib/rack_proxy_examples/`. To mount any example into an existing Rails app:

1. create `config/initializers/proxy.rb`
2. modify the file to require the example file
```ruby
require 'rack_proxy_examples/forward_host'
```

### Forward request to Host and Insert Header

Test with `require 'rack_proxy_examples/forward_host'`

```ruby
class ForwardHost < Rack::Proxy

  def rewrite_env(env)
    env["HTTP_HOST"] = "example.com"
    env
  end

  def rewrite_response(triplet)
    status, headers, body = triplet

    # example of inserting an additional header
    headers["X-Foo"] = "Bar"

    # if you rewrite env, it appears that content-length isn't calculated correctly
    # resulting in only partial responses being sent to users
    # you can remove it or recalculate it here
    headers["content-length"] = nil

    triplet
  end

end
```

### Disable SSL session verification when proxying a server with e.g. self-signed SSL certs

Test with `require 'rack_proxy_examples/trusting_proxy'`

```ruby
class TrustingProxy < Rack::Proxy

  def rewrite_env(env)
    env["HTTP_HOST"] = "self-signed.badssl.com"

    # We are going to trust the self-signed SSL
    env["rack.ssl_verify_none"] = true
    env
  end

  def rewrite_response(triplet)
    status, headers, body = triplet

    # if you rewrite env, it appears that content-length isn't calculated correctly
    # resulting in only partial responses being sent to users
    # you can remove it or recalculate it here
    headers["content-length"] = nil

    triplet
  end

end
```

The same can be achieved for *all* requests going through the `Rack::Proxy` instance by using

```ruby
Rack::Proxy.new(ssl_verify_none: true)
```

### Rails middleware example

Test with `require 'rack_proxy_examples/example_service_proxy'`

```ruby
###
# This is an example of how to use Rack-Proxy in a Rails application.
#
# Setup:
# 1. rails new test_app
# 2. cd test_app
# 3. install Rack-Proxy in `Gemfile`
#    a. `gem 'rack-proxy', '~> 0.7.4'`
# 4. install gem: `bundle install`
# 5. create `config/initializers/proxy.rb` adding this line `require 'rack_proxy_examples/example_service_proxy'`
# 6. run: `SERVICE_URL=http://guides.rubyonrails.org rails server`
# 7. open in browser: `http://localhost:3000/example_service`
#
###
ENV['SERVICE_URL'] ||= 'http://guides.rubyonrails.org'

class ExampleServiceProxy < Rack::Proxy
  def perform_request(env)
    request = Rack::Request.new(env)

    # use rack proxy for anything hitting our host app at /example_service
    if request.path =~ %r{^/example_service}
        backend = URI(ENV['SERVICE_URL'])
        # most backends required host set properly, but rack-proxy doesn't set this for you automatically
        # even when a backend host is passed in via the options
        env["HTTP_HOST"] = backend.host

        # This is the only path that needs to be set currently on Rails 5 & greater
        env['PATH_INFO'] = ENV['SERVICE_PATH'] || '/configuring.html'

        # don't send your sites cookies to target service, unless it is a trusted internal service that can parse all your cookies
        env['HTTP_COOKIE'] = ''
        super(env)
    else
      @app.call(env)
    end
  end
end
```

### Using as middleware to forward only some extensions to another Application

Test with `require 'rack_proxy_examples/rack_php_proxy'`

Example: Proxying only requests that end with ".php" could be done like this:

```ruby
###
# Open http://localhost:3000/test.php to trigger proxy
###
class RackPhpProxy < Rack::Proxy

  def perform_request(env)
    request = Rack::Request.new(env)
    if request.path =~ %r{\.php}
      env["HTTP_HOST"] = ENV["HTTP_HOST"] ? URI(ENV["HTTP_HOST"]).host : "localhost"
      ENV["PHP_PATH"] ||= '/manual/en/tutorial.firstpage.php'

      # Rails 3 & 4
      env["REQUEST_PATH"] = ENV["PHP_PATH"] || "/php/#{request.fullpath}"
      # Rails 5 and above
      env['PATH_INFO'] = ENV["PHP_PATH"] || "/php/#{request.fullpath}"

      env['content-length'] = nil

      super(env)
    else
      @app.call(env)
    end
  end

  def rewrite_response(triplet)
    status, headers, body = triplet

    # if you proxy depending on the backend, it appears that content-length isn't calculated correctly
    # resulting in only partial responses being sent to users
    # you can remove it or recalculate it here
    headers["content-length"] = nil

    triplet
  end
end
```

To use the middleware, please consider the following:

1) For Rails we could add a configuration in `config/application.rb`

```ruby
  config.middleware.use RackPhpProxy, {ssl_verify_none: true}
```

2) For Sinatra or any Rack-based application:

```ruby
class MyAwesomeSinatra < Sinatra::Base
   use  RackPhpProxy, {ssl_verify_none: true}
end
```

This will allow to run the other requests through the application and only proxy the requests that match the condition from the middleware.

See tests for more examples.

### SSL proxy for SpringBoot applications debugging

Whenever you need to debug communication with external services with HTTPS protocol (like OAuth based) you have to be able to access to your local web app through HTTPS protocol too. Typical way is to use nginx or Apache httpd as a reverse proxy but it might be inconvinuent for development environment. Simple proxy server is a better way in this case. The only what we need is to unpack incoming SSL queries and proxy them to a backend. We can prepare minimal set of files to create autonomous proxy server.

Create `config.ru` file:
```ruby
#
# config.ru
#
require 'rack'
require 'rack-proxy'

class ForwardHost < Rack::Proxy
  def rewrite_env(env)
    env['HTTP_X_FORWARDED_HOST'] = env['SERVER_NAME']
    env['HTTP_X_FORWARDED_PROTO'] = env['rack.url_scheme']
    env
  end
end

run ForwardHost.new(backend: 'http://localhost:8080')
```

Create `Gemfile` file:
```ruby
source "https://rubygems.org"

gem 'thin'
gem 'rake'
gem 'rack-proxy'
```

Create `config.yml` file with configuration of web server `thin`:
```yml
---
ssl: true
ssl-key-file: keys/domain.key
ssl-cert-file: keys/domain.crt
ssl-disable-verify: false
```

Create 'keys' directory and generate SSL key and certificates files `domain.key` and `domain.crt`

Run `bundle exec thin start` for running it with `thin`'s default port.

Or use `sudo -E thin start -C config.yml -p 443` for running with default for `https://` port.

Don't forget to enable processing of `X-Forwarded-...` headers on your application side. Just add following strings to your `resources/application.yml` file.
```yml
---
server:
  tomcat:
    remote-ip-header: x-forwarded-for
    protocol-header:  x-forwarded-proto
  use-forward-headers:  true
```

Add some domain name like `debug.your_app.com` into your local `/etc/hosts` file like
```
127.0.0.1	debug.your_app.com
```

Next start the proxy and your app. And now you can access to your Spring application through SSL connection via `https://debug.your_app.com` URI in a browser.

### Using SSL/TLS certificates with HTTP connection
This may be helpful, when third-party API has authentication by client TLS certificates and you need to proxy your requests and sign them with certificate. 

Just specify Rack::Proxy SSL options and your request will use TLS HTTP connection:
```ruby
# config.ru
. . .

cert_raw = File.read('./certs/rootCA.crt')
key_raw = File.read('./certs/key.pem')

cert = OpenSSL::X509::Certificate.new(cert_raw)
key = OpenSSL::PKey.read(key_raw)

use TLSProxy, cert: cert, key: key, use_ssl: true, verify_mode: OpenSSL::SSL::VERIFY_PEER, ssl_version: 'TLSv1_2'
```

And rewrite host for example:
```ruby
# tls_proxy.rb
class TLSProxy < Rack::Proxy
  attr_accessor :original_request, :query_params

  def rewrite_env(env)
    env["HTTP_HOST"] = "client-tls-auth-api.com:443"
    env
  end
end
```

WARNING
----

Doesn't work with `fakeweb`/`webmock`. Both libraries monkey-patch net/http code.

Todos
----

* Make the docs up to date with the current use case for this code: everything except streaming which involved a rather ugly monkey patch and only worked in 1.8, but does not work now.
* Improve and validate requirements for Host and Path rewrite rules
* Ability to inject logger and set log level
