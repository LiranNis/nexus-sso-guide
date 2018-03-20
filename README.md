# Nexus SSO Guide - WIP

## Steps
1. **Configure LDAP**  
- Login to Nexus with admin
- go to Security > LDAP
- fill it with the right information, my advice is to create user that will serve only this purpose, for example: Nexus-Auth  
![Creating LDAP connection](img/ldap.PNG?raw=true "Creating LDAP connection")  
Verify the connection and press next.  
- customize the user and group settings, in this example I removed the Base DN and selected the `User subtree` to include the entire users in the AD, you can press `Verify user mapping` in order to check which users included in the filter.  
![LDAP user and group settings](img/ldap2.png?raw=true "LDAP user and group settings")
2. **HTTPD**  
Choose the server that you want to use as your proxy and install httpd on it, I use the nexus server itself.  
`yum install httpd` (or another package manager)
3. **Create HTTP keytab for nexus**
- We will register the nexus HTTP SPN to a user, create a user for that purpose, again my advice is a new user, for example: Nexus-HTTP
- Create the SPN  
`setspn -S HTTP/<hostname>.<domain> <new-user>`  
If there is duplicate you will get an error containing the user that the SPN is registered to, use:  
`setspn -D HTTP/<hostname>.<domain> <old-user>` to delete the spn and then run the first command again.
- Create keytab - to create the keytab run the following command on Windows machine and enter your password
```
ktpass -princ HTTP/<ServerFQDN>@<Domain> -pass * -mapuser Nexus-Auth@<Domain> -pType KRB5_NT_PRINCIPAL -crypto all -out "C:\http.keytab"
```
- Move the keytab to /etc/httpd/http.keytab in your httpd server (if you choose another location you will need to change the keytab location in the next stage)
4. **Configure HTTPD**
Create
- Install httpd    
  ```yum install httpd```
- Edit `conf/httpd.conf` and add:  
  ``` 
  <Location />
  #Configure basic authentication (for testing purposes)
  AuthType Basic
  AuthName "Sonatype Nexus"
  AuthBasicProvider file
  AuthUserFile /home/test/passwd
  Require valid-user
  
  # Make REMOTE_USER set by authentication available as environment variable
  RewriteEngine on
  RewriteCond %{REMOTE_USER} (.*)
  RewriteRule .* - [E=ENV_REMOTE_USER:%1]
  RequestHeader set X-Proxy-REMOTE-USER %{ENV_REMOTE_USER}e
  
  # Remove incoming authorization headers, Nexus users are authenticated by HTTP header
  RequestHeader unset Authorization
  
  # Configure apache as a reverse proxy for Nexus
  ProxyPreserveHost on
  ProxyPass http://localhost:8081/nexus
  ProxyPassReverse http://localhost:8081/nexus
  </Location>
  ```
  
  
## References
- [How to Configure Request Header Authentication in Nexus with Apache](https://support.sonatype.com/hc/en-us/articles/214942368-How-to-Configure-Request-Header-Authentication-in-Nexus-with-Apache)
- [Configure Apache to use Kerberos authentication](http://www.microhowto.info/howto/configure_apache_to_use_kerberos_authentication.html)
- [Rundeck SSO guide](https://github.com/genadipost/rundeck-sso-guide)
