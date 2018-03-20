# Nexus SSO Guide - WIP

## Steps
1. **Configure LDAP**  
Login to Nexus with admin and go to Security > LDAP, and fill it with the right information.  
My advice is to create user that will serve only this purpose, for example: Nexus-Auth
![Creating LDAP connection](img/ldap.PNG?raw=true "Creating LDAP connection")

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
