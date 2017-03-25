+++
date = "2017-03-24T21:35:40Z"
description = ""
title = "Using HA Proxy for SaaS systems - Custom Domains"

+++

<style>
.hljs {
    white-space: pre;
    overflow-x: auto;
}
</style>

A common problem with setting up a SaaS system is how to manage the customers on a custom domain.

It's easy when everybody enters your system through `example.com`, and even `account-name.example.com` wildcards, but what if your system offers the ability to have a custom domain such as `zando.io`.

This is a problem we had at Shopblocks, and I tried a few ways of solving it. 

---

If you are experienced with HA Proxy and just want to see the configuration, skip to the bottom.

If you want to experience the journey with me, read on!

---

The first involved using Ansible to create a new virtual host for Apache and then reloading the apache server. 

This worked, however, performance became an issue pretty quickly. 

Registrations were taking a long time, sometimes up to a minute.

Using Ansible with Apache also had no guarantee of consistency, if a server was offline during the registration process then that server didn't have the virtual host, and could not resolve the request.

To fix the consistency issue we sharing the virtual hosts across the platform on a mounted drive using NFS. The performance of the mounted drive was not majorly affected by this move, but the performance was still an issue on the process side. We had solved the consistency issue, but we still had a couple of others.

Because we enforce HTTPS on all requests, our load balancer could not utilise proxying with the clients IP address. This meant that our logs from Apache, and internally in our system, all would have the load balancers private IP and not the clients IP.

We also still had the performance issue. This wasn't really acceptable, and so we wanted a completely different approach.

---

We were already using HA proxy for our load balancing, and we knew that HA proxy could terminate SSL connections and modify requests. 

If we could solve:

- SSL connection terminating
- Retaining the client IP address
- Unique Virtual Hosts
- Use of Ansible

Then we should be in a much better position.

It would allow us to more easily perform upgrades on our application servers.

The possibilities for upgrades were plentiful; whether that be changing from Apache to Nginx, adding more servers to the clusters easily or even moving to completely different architecture in the backend (for example, NodeJS instead of Apache + PHP).

These sorts of upgrades are now much simpler to perform.

It seemed like we were going down the correct route.

---

The first problem to solve is forcing HTTP to HTTPS redirects. This is very easy to do in HA Proxy

```ini
frontend http
    bind :80

    redirect scheme https code 301

frontend https
    bind :443

    # We will fill this in later
```

We are listening for requests on port 80 and port 443.

Any request that comes in on port 80 will be redirected with a `301 Permanent Redirect` to `https`.

We are now done with any HTTP requests. Everything from now on is HTTPS only.

---

Now that all of our traffic is HTTPS, we need to terminate those requests.

At Shopblocks, all new accounts start with a couple of `myshopblocks.com` subdomains. 

We have a wildcard SSL certificate available that covers all of these domains.

Customers can also setup a custom domain. This gets provided with a Lets Encrypt certificate.

Allowing Lets Encrypt certificates in this setup is a topic of its own, but the gist is you need to get the certificate in `pem` format, which involves concatonating the `private key` and `ceritifcate` together into a flat directory file.

```ini
frontend https
    bind :443 ssl crt /path/to/certificates

    # We will fill this in later
```

This directory of certificates must be flat, contain at least 1 certificate that is valid, and be in `.pem` format. The filename of the certificate does not matter.

Note, that this method uses SNI headers from the request, and so some requests from old browsers, for example Android 2.3, may not return with the correct certificate, and instead take the first certificate it finds. This is not a major issue, since the systems that do not use SNI are largely outdated and less commonly used.

---

Now that all of our traffic is HTTPS and is being terminated, we need to load balance the requests and send them through to our web servers.

```ini
frontend https
    bind :443 ssl crt /path/to/certificates

    use_backend core

backend core
    balance roundrobin

    server WEB_HOSTNAME_01 10.20.30.40:80 check
    server WEB_HOSTNAME_02 10.20.30.41:80 check
```

Here we have 2 servers defined in our backend named `core`. The backend name does not matter, and is used as an identifier between the `use_backend` and the backend itself.

The two servers are being referenced using a private network IP. You can use either public or private network IP's, however I would suggest private networking so that you can have finer control and limit direct access to the server.

Because of the SSL has already been terminated in the load balancer, we now have a plain HTTP request which we will send through on port 80 to our web server.

---

Our traffic is now going through the load balancer and hitting a web server, however you are probably getting the 000-default page in apache. 

The reason for this is that your request is still coming through on `zando.io` instead of a domain name that the apache server knows how to resolve.

To fix this issue we need to modify the headers sent through.

This has some caveats, such as not being able to use the HTTP_HOST server variables in PHP as expected. There are ways around that, but it is worth keeping in mind.

I am going to map the request through to `system.example.com` in this example. In reality you can set this to any domain, whether you control it or not since no DNS lookups take place.

```ini
backend core
    ...

    http-request set-header Host system.example.com

    ...
```

The domain you choose to replace `system.example.com` needs to be recognised by your apache virtual host on port 80.

---

You are now getting the correct page, but your system does not know which domain was originally requested, and by extension, what the customer's ID is.

To fix this, we will need a map of all domains to a customer ID.

This is stored as a simple key => value pair in a text file.

```ini
domain1.com 1
domain2.com 2
domain3.com 3
```

`domain1.com` belongs to customer with an ID of `1`, and so on.

This file is going to be referred to as our `customer.map` file.

Whenever a new customer or domain gets added to your system, you simply need to append to this file, and reload haproxy.

```ini
frontend https
    ...

    http-request set-header X-Customer-ID %[req.hdr(host),lower,map_str(/path/to/customer.map)]

    ...
```

This will set the `X-Customer-ID` header based on the `value` matching the `host`. You want to ensure that your `customer.map` file is all lowercase, since the `lower` function in use will convert the request header to lowercase.

---

We now have the system assigning the `X-Customer-ID` for domains that the map file contains, but if it does not contain a file, the `X-Customer-ID` is set, but empty. 

If this is okay in your system, then leave it as it is, however if you do not want to resolve requests at all for any domains you do not recognise then above the `http-request` line, you can add the following.

```ini
frontend https
    ...

    acl is_customer hdr(host),lower,map_str(/path/to/customer.map) -m found

    http-request silent-drop unless is_customer

    http-request set-header ...

    ...
```

Now the connection will drop immediately for any domain requests not recognised in the `customer.map` file.

We determine whether somebody is a customer using an [Access Control List (acl)](https://en.wikipedia.org/wiki/Access_control_list). 

This will create a variable called `is_customer` which will be a variable based on the `found` method specified by the `-m` flag used.

We can assume that if the domain is found in the list, they are a customer of ours.

---

Putting that all together you should have something that looks like the following

```ini
frontend http
    bind :80

    redirect scheme https code 301

frontend https
    bind :443 ssl crt /path/to/certificates

    acl is_customer hdr(host),lower,map_str(/path/to/customer.map) -m found

    http-request silent-drop unless is_customer

    http-request set-header X-Customer-ID %[req.hdr(host),lower,map_str(/path/to/customer.map)]

    use_backend core

backend core
    balance roundrobin

    http-request set-header Host system.example.com

    server WEB_HOSTNAME_01 10.20.30.40:80 check
    server WEB_HOSTNAME_02 10.20.30.41:80 check 
```

And that is the basics of setting up a SaaS resolver in HA proxy and passing it through to Apache.

If you have any questions, feel free to send me a message on [twitter](https://twitter.com/bowersbros) or [email me](mailto:bowersbros+haproxy_blog_post@gmail.com)

Thank you to [Hugojmd](https://twitter.com/hugojmd) for proof reading and spotting mistakes.