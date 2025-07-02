## Reverse Proxy
### Problem: 
Our containers are accessible from the server's hostname on the ports that we choose to specify when running. 

> See *[Starting an App in a Container](https://github.com/levy013/podman-research?tab=readme-ov-file#step-4-start-app-in-container)*

A reverse proxy will allow us to specify a server name and endpoint for users to reach out to

Here is an example for resolving http://uit1446:8080/scalar under http://cadlnx/mq/scalar

1. Install Nginx
sudo dnf install nginx
sudo systemctl enable --now nginx
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload
2. Set up custom conf inside /etc/nginx/conf.d/
3. Add RP logic
```nginx
server { 
        listen 80; 
        server_name cadlnx; 
         
        location /mq/ { 
                rewrite ^/mq/(.*)$ /$1 break; 
                proxy_pass http://127.0.0.1:8080;

                # generic reverse proxy set_header directives 
                proxy_set_header Host $host; 
                proxy_set_header X-Real-IP $remote_addr; 
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
                proxy_set_header X-Forwarded-Proto $scheme; 
        } 
} 
```
4. Update SELinux policy? Check via getsebool httpd_can_network_connect
sudo setsebool -P httpd_can_network_connect 1
OR 
sudo setsebool -P httpd_can_network_relay 1
sudo systemctl reload nginx

Look into this more
