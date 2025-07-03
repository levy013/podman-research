## Installing Nginx
`sudo dnf install nginx`

`sudo systemctl enable --now nginx`

## Configure the `firewalld` service to accept http requests
🔗[RHEL firewalld Docs](https://www.redhat.com/en/blog/firewalld-linux-firewall)

By default, `firewalld` does not allow http requests

`sudo firewall-cmd --add-service=http --permanent`

`sudo firewall-cmd --reload`

## SELinux
🔗 [RHEL SELinux Docs](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/using_selinux/index)

When SELinux Enforcing mode is enabled, the policy for `httpd_t` (which Nginx runs under) restricts outbound network connections by default. 

This means that Nginx is not allowed to make outbound requests, e.g. using `proxy_pass`:

`connect() to 127.0.0.1:8080 failed (13: Permission denied)`

Instead of setting SELinux to permissive mode, we can simply allow Nginx to make outbound connections:

`sudo setsebool -P httpd_can_network_connect on`
 
