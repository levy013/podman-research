## Reverse Proxy

🔗 *[Starting an App in a Container](https://github.com/levy013/podman-research?tab=readme-ov-file#step-4-start-app-in-container)*

🔗 *[Installing Nginx]()*

### Use Case: 
Our containers are accessible from the server's hostname on the ports that we choose to specify when running. 
We've set up our server to listen for requests matching a particular shape so that we can forward it to a particular destination.
A reverse proxy will allow us to specify a server name and endpoint for users to reach out to

Here is an example for setting up CAD.MQ 
- Our application is running on the server under http://localhost:8080/ 
- Clients can make requests to http://uit1446:8080/
- We want them to be able use a more accessible URI, such as http://cadlnx/mq/

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
📝 A note about `rewrite ^/mq/(.*)$ /$1 break;`
> *It's important to remember that we're simply proxying traffic here. The idea behind this rewrite is that we need to strip /mq/ from the URI and reappend any parameters from the capture group.*
>
> Again, we're essentially just creating a pretty URL for the client to leverage.
>
>    - the client sees http://cadlnx/mq/scalar
>    - the server sees http://127.0.0.1:8080/scalar
>
> The regex is relatively straightforward as well
>    - ^/mq/ : match only if we're at the beginning of the URI, so http://cadlnx/mq will match but http://cadlnx/foo/mq will not
>    - (.*) : wildcard for capture group
>    - $ : anchor to match end of string
>    - /$1 : replacement to append parameters from capture group to '/'
