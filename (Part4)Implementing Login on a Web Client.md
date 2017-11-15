# Implementing Login on a Web Client


If we want to use our current setup to authenticate users on a client, like a website, we want them to be able to call a login method on our service. Currently, we would need to call a service methods, and catch and unauhtorized exception if the credentials are invalid. We could implement an ```IAuthService``` Interface with a Login method (aswell as our ```IService``` interface) on our ```Service1```, but if we make a habbit of this, the size of our service implementation will grow, as the application grows, so i prefer seperating our authentication into its own little service.

# Step 1 - Setting up another service for authentication
  - Create an Interface in your WCF Service project, called IAuthService (see below)
```c#
[ServiceContract]
public interface IAuthService
{
    [OperationContract]
    bool Login(string username, string password);
}
```
# Step 2 - Implement this interface in a class you create in your WCF service project, called AuthService
```c#
public class AuthService : IAuthService
{
    public bool Login(string username, string password)
    {
        return "SuperStudent" == username && "1234" == password;
    }
}
```
# Step 3 - Configuring your Service Host 
Because we have added another service, we need to set it up in the ```<system.serviceModel>``` config. We of course also need to configure the endpoints we want for this new ```AuthService```.

  - When we add another service, we need to add a base address for this service aswell. This means it will be an independantly hosted service, and this it must have a different port than our first service. I assigned the port ```9978```. Since we only bound our X-509 certificate to port ```9977```, you need to bind the certificate to this new port aswell (you can see the command in part 1).
  - Notice ```behaviorConfiguration="SecureBevaiorAuth"```. This new service uses a different ```<serviceBehavior>``` than your first service, we need to set this up in a minute.

Add the following to your <system.serviceModel>

<service name="ProjectName.AuthService"  behaviorConfiguration="SecureBevaiorAuth">
  <host>
    <baseAddresses>
      <add baseAddress="https://localhost:9978/"   />
      <!--Remember to bind your certificate to the new port 9978-->
    </baseAddresses>
  </host>
  <endpoint address="SecureAuthService" binding="wsHttpBinding" contract="ProjectName.IAuthService" bindingConfiguration="SecureAuthEndpoint"/>

  <endpoint address="mex" binding="mexHttpsBinding" contract="IMetadataExchange"/>
  <endpoint address="mexTcp" binding="mexTcpBinding" contract="IMetadataExchange"/>
</service>

