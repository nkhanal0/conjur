---
title: Tutorial - Ruby
layout: page
---

{% include toc.md key='introduction' %}

The [Conjur API for Ruby](https://github.com/conjurinc/api-ruby) provides a robust programmatic interface to Conjur. You can use the Ruby API to authenticate with Conjur, load policies, fetch secrets, perform permission checks, and more.

{% include toc.md key='prerequisites' %}

* A [Conjur server](/conjur/installation/server.html) endpoint.
* The [Conjur API for Ruby](https://github.com/conjurinc/api-ruby), version 5.0 or later.

{% include toc.md key='setup' %}

To demonstrate the usage the Conjur API, we need some sample data loaded into the server.

Save this file as "conjur.yml":

{% include policy-file.md policy='api_integration' %}

It defines:

* `variable:db/password` Contains the database password.
* `webservice:backend`
* `layer:myapp` A layer (group of hosts) with access to the password and the webservice.

Load the policy using the following command:

{% highlight shell %}
$ conjur policy load --replace bootstrap conjur.yml
Loaded policy 'bootstrap'
{
  "created_roles": {
    "dev:host:myapp-01": {
      "id": "dev:host:myapp-01",
      "api_key": "1wgv7h2pw1vta2a7dnzk370ger03nnakkq33sex2a1jmbbnz3h8cye9"
    }
  },
  "version": 1
}
{% endhighlight %}

Now, use OpenSSL to generate a random secret, and load it into the database password variable:

{% include db-password.md %}

{% include toc.md key='configuration' %}

The Ruby API is configured using the `Conjur.configuration` object. The most important options are:

* `appliance_url` The URL to the Conjur server.
* `account` The Conjur organization account name.

Create a new Ruby program, require the `conjur-api` library, and set these two parameters in the following manner:

{% highlight ruby %}
irb(main)> require 'conjur-api'
irb(main)> Conjur.configuration.appliance_url = "https://possum-ci-conjur.herokuapp.com"
irb(main)> Conjur.configuration.account = "dev" # <- REPLACE ME!
{% endhighlight %}

<div class="note">
<strong>Note</strong> Configuration can also be provided via environment variables. The environment variable pattern is <tt>CONJUR_&lt;setting></tt>. For example, <tt>CONJUR_APPLIANCE_URL=https://possum-ci-conjur.herokuapp.com</tt>
</div>

{% include toc.md key='authentication' %}

Once the server connection is configured, the next step is to authenticate to obtain an access token. When you create a Conjur Host, the server issues an API key which you can use to authenticate as that host. Here's how you use it in Ruby:

{% highlight ruby %}
irb(main)> host_id = "host/myapp-01"
irb(main)> api_key = "1vgw4jzvyzmay95mrx2s5ad1d28gt3gh2gesb1411kqcah3nrv01r"
irb(main)> conjur = Conjur::API.new_from_key host_id, api_key
irb(main)> puts conjur.token
{"data"=>"admin", "timestamp"=>"2017-06-01 13:26:59 UTC", "signature"=>"NBc5...a7LLJl", "key"=>"ccd789173e1fc4770ac66cd1acf498b4"}
{% endhighlight %}

<div class="note">
<strong>Note</strong> Authentication credentials can also be provided via environment variables. Use <tt>CONJUR_AUTHN_LOGIN</tt> for the login name, and
<tt>CONJUR_AUTHN_API_KEY</tt> for the API key.
</div>

{% include toc.md key='secrets_fetching' %}

Once authenticated, the API client can be used to fetch the database password:

{% highlight ruby %}
irb(main)> variable = conjur.resource("#{Conjur.configuration.account}:variable:db/password")
irb(main)> puts variable.value
ef0a4822539369659fbfb267
{% endhighlight %}

{% include toc.md key='permission_checking' %}

To check a permission, load the Conjur resource (typically a Webservice) on which the permission is defined.

Then use the `permitted?` method to test whether the Conjur user has a specified privilege on the resource.

In this example, we determine that `host:myapp-01` is permitted to `execute` but not `update` the resource `webservice:backend`:

{% highlight ruby %}
irb(main)> webservice = conjur.resource("#{Conjur.configuration.account}:webservice:backend")
irb(main)> puts webservice.permitted? 'execute'
true
irb(main)> puts webservice.permitted? 'update'
false
{% endhighlight %}
