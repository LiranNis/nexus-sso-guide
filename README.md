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
ktpass -princ HTTP/<ServerFQDN>@<Domain> -pass * -mapuser Nexus-HTTP@<Domain> -pType KRB5_NT_PRINCIPAL -crypto all -out "C:\http.keytab"
```
**Note:** this command tends to break users, so you can't login to it anymore, ensure you use user that you don't need to login!
- Move the keytab to /etc/httpd/http.keytab in your httpd server (if you choose another location you will need to change the keytab location in the next stage)
4. **Create Rut Auth capability**
- Login to nexus with admin
- Go to System > Capabilities
- Click Create capability
- Choose Rut Auth
- Ensure `Enable this capability` marked and insert `X-Proxy-REMOTE-USER` to the HTTP Header name
- Click Create capability
5. **Configure HTTPD**  
- Install `mod_auth_gssapi`  
```
yum install mod_auth_gssapi
```
- Create `gssapi.conf` under `/etc/httpd/conf.d` with this content  
```
LoadModule auth_gssapi_module modules/mod_auth_gssapi.so

<VirtualHost *:80>
        ServerName centosnexus
        <Location "/">
                AuthType GSSAPI
                AuthName "Kerberos Authentication"
                GssapiCredStore keytab:/etc/httpd/http.keytab
                RewriteEngine On
                RewriteCond %{LA-U:REMOTE_USER}(.+(?=@))
                RewriteRule .-[E=RU:%1]
                RequestHeader set X-Remote-User "%{RU}e" env=RU
                Require valid-user
                ProxyPreserveHost on
                ProxyPass http://centosnexus:8081
                ProxyPassReverse http://centosnexus:8081
        </Location>
</VirtualHost>

```

  
## References
- [How to Configure Request Header Authentication in Nexus with Apache](https://support.sonatype.com/hc/en-us/articles/214942368-How-to-Configure-Request-Header-Authentication-in-Nexus-with-Apache)
- [Configure Apache to use Kerberos authentication](http://www.microhowto.info/howto/configure_apache_to_use_kerberos_authentication.html)
- [Rundeck SSO guide](https://github.com/genadipost/rundeck-sso-guide)
