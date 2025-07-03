# Reverse Proxy

🔗 *[Starting an App in a Container](https://github.com/levy013/podman-research?tab=readme-ov-file#step-4-start-app-in-container)*

🔗 *[Installing Nginx]()*

## What is a reverse proxy?
*A reverse proxy acts as an intermediary between clients and servers. They intercept client requests and forward them to the appropriate backend service.*
https://www.cloudflare.com/learning/cdn/glossary/reverse-proxy/

## Use Case: 
By default, our applications are accessible from the server's hostname on the ports that we choose to expose when running the containers.
e.g. we expose port 8080 on uit1446 to serve CAD.MQ.API => http://uit1446:8080/api/list

But this is admittedly pretty obnoxious for the client to use effectively.

Assuming we've set up some DNS entries on the local network (or modified our Hosts file), we can use Nginx to bind our apps to those addresses on the server and serve our content from an "alias".

A reverse proxy will allow us to specify both a hostname and it's related path(s) for the client to reach out to. This maintains a consistent "shape" for the request to take on that Nginx can subsequently match against and proxy to the corresponding resource on the server.

Here is an example using CAD.MQ.API
- Our application is running on the server exposed on http://localhost:8080/ 
- Clients can natively make requests to the service on http://uit1446:8080/
- With a reverse proxy we can offer the client a prettier URI, such as http://cadlnx/mq/

## Setup:
1. Set up custom conf inside of `/etc/nginx/conf.d/`
>  Instead of modifying `/etc/nginx/nginx.conf` directly, it's generally considered best practice to create a custom conf inside of `/conf.d`
>
> In this example, we're writing to `/etc/nginx/conf.d/cad.conf`

2. Add the reverse proxy config
```nginx
server { 
        listen 80; # what port Nginx should listen on
        server_name cadlnx; # what hostname the following location block(s) should respond to
         
        location /mq/ { # location block containing rules/logic for handling requests to http://cadlnx/mq/*
                rewrite ^/mq/(.*)$ /$1 break; 
                proxy_pass http://127.0.0.1:8080; # underlying address we're proxying traffic to 

                proxy_set_header Host $host; # retains original host value e.g. cadlnx
                proxy_set_header X-Real-IP $remote_addr; # client's real IP addr 
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; # full chain of fwd IPs
                proxy_set_header X-Forwarded-Proto $scheme; # http or https 
        }

        location /foo/ { # e.g. some other location block for handling http://cadlnx/foo/
                ...
        }
} 
```
📝 Understanding `rewrite ^/mq/(.*)$ /$1 break;`
> *It's important to remember that we're simply proxying traffic here. The idea behind this rewrite is that we need to strip /mq/ from the URI and reappend any parameters from the capture group.*
>
> Again, we're essentially just creating a pretty URL for the client to leverage.
>    - The client requests http://cadlnx/mq/scalar
>    - Nginx intercepts and massages the request
>    - The server responds with the resource found at http://127.0.0.1:8080/scalar
>
> The regex is relatively straightforward as well
>    - ^/mq/ : match only if we're at the beginning of the URI, so http://cadlnx/mq will match but http://cadlnx/foo/mq will not
>    - (.*) : wildcard for capture group
>    - $ : anchor to match end of string
>    - /$1 : replacement to append parameters from capture group to '/'
