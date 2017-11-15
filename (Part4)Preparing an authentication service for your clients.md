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

  - When we add another service, we need to add a ```<baseAddress>``` for this service aswell. This means it will be an independantly hosted service, and this it must have a different port than our first service. I assigned the port ```9978```. Since we only bound our X-509 certificate to port ```9977```, you need to bind the certificate to this new port aswell (you can see the command in part 1).
  - Notice ```behaviorConfiguration="SecureBevaiorAuth"```. This new service uses a different ```<serviceBehavior>``` than your first service, we need to set this up in a minute.

  - Before we do the new ```<behavior>``` configuration, add the following to your <system.serviceModel>:
```xml
<service name="ProjectName.AuthService"  behaviorConfiguration="SecureBevaiorAuth">
  <host>
    <baseAddresses>
      <add baseAddress="https://localhost:9978/"   />
      <!--Remember to bind your certificate to the new port 9978-->
    </baseAddresses>
  </host>
  <endpoint address="SecureAuthService" binding="wsHttpBinding" contract="ProjectName.IAuthService" bindingConfiguration="SecureAuthEndpoint"/>
  <endpoint address="mex" binding="mexHttpsBinding" contract="IMetadataExchange"/>
</service>
```
# Step 4 - Configuring the behavior of the AuthService
- Go to your <behaviors> --> <serviceBehaviors> and add another <behavior> inside this element.
```xml
<behavior name="SecureBevaiorAuth">
  <serviceCredentials>
    <serviceCertificate x509FindType="FindByThumbprint" findValue="86D979B0F41A65D806638558B7C09EDADFD753D8" storeName="My" storeLocation="LocalMachine" />
  </serviceCredentials>
  <serviceMetadata httpGetEnabled="False" httpsGetEnabled="True"/>
  <serviceDebug includeExceptionDetailInFaults="True" />
</behavior>
```
Notice this element is very similar to your current ```wsHttpBinding``` configuration. The difference here is, that we have removed the ```<userNameAuthentication>``` and the ```<serviceAuthorization>``` to disable credential checking. These are not needed, as we want unauthorized users to be able to call the Login method, to find out if they are allowed in with their credentials.
  
 # Step 5 - Configuring the AuthService endpoint
 Finally, because we have decided to remove the credential validation from the AuthService, we cannot use the currently existing endpoint configuration called SecureEndpoint. Go to <bindings> --> <wsHttpBinding> and add the following.
```xml
<binding name="SecureAuthEndpoint">
  <security mode="Transport">
    <transport clientCredentialType="None"/>
  </security>
</binding>
```
Notice that the ```<security mode="Transport">``` is different than the other binding configuration. This time we speficy that the security mode is transport, and that message credentials isn't needed. This means that we don't encrypt the message or use username/password validation. But our connection is still secure, since we have transport security, and a certificate specified in the ```<behavior>``` configuration.
  
# Step 6 - Hosting the new service.
Go your Console hosting applications Program.cs, and change the Main method as follows:
```c#
static void Main(string[] args)
{
    ServiceHost host = new ServiceHost(typeof(IService1));
    ServiceHost authhost = new ServiceHost(typeof(AuthService));
    host.Open();
    Console.WriteLine("Service1 host is running...");
    authhost.Open();
    Console.WriteLine("AuthService host is running");
    Console.ReadLine();//Keep program running.
    host.Close();
    authhost.Close();

}
```
# Step 7 - Testing it all out
  - Go to your Console Client and add a service reference to the new AuthService (using the ```<baseAddress>``` from your config)
  - Instantiate a 
  ```AuthServiceClient authClient = new AuthServiceClient();```
  - And call the Login method on your service.
  - If the service returns true, your credentials were validated, and you can now use them to call your Username and password secured service from earlier. It could look something like this:
```c#
AuthServiceClient authClient = new AuthServiceClient();
var isLoggedIn = authClient.Login("SuperStudent", "1234");
if (isLoggedIn)
{
    SecureUserServiceClient client = new SecureUserServiceClient("WSHttpBinding_ISecureUserService");
    client.ClientCredentials.UserName.UserName = "SuperStudent";
    client.ClientCredentials.UserName.Password = "1234";
    var data = client.GetData(1337);
}
```
Run your program and test it out.

### Whenever you call the Login method on the Auth service, you can store the credentials on your client, and pass them along to all of your Service1 calls.

This concludes part 4 of the tutorial series
