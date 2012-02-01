#**Bodiam**

##*Installation and Basic Usage*

#**Overview**
Bodiam is an authorization library that restricts what items a given user is allowed to access. It allows coarse-grained access checks based on “item types” or fine-grained to specific instances of the items. Permissions can be granted to users or roles. Enforcement is managed with an API or using ASP.NET MVC attributes. The current implementation expects the application to use federated authentication.   

#**Installation**   
##*Installation Requirements*   
- Microsoft® Windows® Vista SP2 (32-bits or 64-bits) , Microsoft® Windows Server 2008 SP2 (32-bit or 64-bit), Microsoft® Windows Server 2008 R2,  Microsoft® Windows®   
- 7 RTM (32-bits or 64-bits)    
- Microsoft® Internet Information Services (IIS) 7.0   
- Microsoft® .NET Framework 4.0   
- Microsoft® Visual Studio 2010   
- Microsoft® Windows Identity Foundation Runtime   
- Microsoft® Windows Identity Foundation SDK 4.0   

**Note**: Bodiam expects the user to use Federated Authentication and a Claim Aware environment. Identity providers are required to issue an IdentityProvider Claim (http://schemas.microsoft.com/accesscontrolservice/2010/07/claims/identityprovider) and a NameIdentifier Claim (http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier)   

#**Setup**  
##Database Configuration   
1. Run the ~/database/Setup-BodiamSchema.cmd script to create the Bodiam schema. Keep in mind that this script will create the tables in an already existing database.

##Aplication Configuration
1. Include the ~/code/Bodiam/Bodiam.csproj project as part of your application’s solution
2. Include the Bodiam project as a reference for your web application project
3. Open the Web.config file of your project
4. Inside the service element in the microsoft.IdentityModel section add the following XML

	```xml
	<microsoft.identityModel>
	  <service>
	     ...
	     <claimsAuthenticationManager type="Bodiam.Web.CustomClaimsAuthenticationManager" />
	    ...
	  </service>
	</microsoft.identityModel>
	```
5. In the conectionStrings section add the connection string to the Bodiam database
    
	```xml
	<configuration>   
    	...  
        <connectionStrings>   
        	...
          <add name="SecurityEntities" 
			connectionString="Data Source=XXXXXXXX;Initial Catalog=XXXXXXXXX;Integrated Security=True" />     
      	...
     	</connectionStrings>   
       	...
     </configuration>
	```
6. Add the SMTP configuration settings for sending invitations to new users
  
    ```xml
	<configuration>
    	...
    	<appSettings>
	        ...
	        <add key="SmtpHost" value="XXXXX" />
	        <add key="SmtpPort" value="XXXXX" />
	        <add key="SmtpUser" value="XXXXX@XXXXX" />
	        <add key="SmtpPassword" value="XXXXX" />
	        <add key="SmtpSenderName" value="XXXXX" />
	        <add key="SmtpSenderAddress" value="XXXXX@XXXXX"   />
	        <add key="InvitationTitle" value="XXXXX"/>
	        ...
	    </appSettings>
        ...
    </configuration>
	```
7. In the appSettings section add the route to where new users will be directed when they receive an invitation 

    ```xml
	<configuration>
		...
		<appSettings>
			...
			<add key="UserAccountInvitationAction" value="/XXXXX/XXXXX" />
			...
		</appSettings>
		...
	</configuration>
    ```
	
##Providing configuration settings values

Application settings and connection strings are retrieved by default from the application configuration file, but you can customize this behavior and retrieve settings from other sources. This can be useful, for instance, if you are hosting your application as a web role in Windows Azure and you want to retrieve settings from the service configuration file.   

The ConfigReader class has two delegates that can be assigned in order to resolve application settings values or connection strings.   

This is an example showing how to provide a delegate to read application settings values from the Windows Azure service configuration file:
   
```c#
    Bodiam.Helpers.ConfigReader.GetConfigValueDelegate = 
	    (string key) =>   
	    {   
	        return RoleEnvironment.GetConfigurationSettingValue(key);   
	    };
```

In a similar way, you can provide a delegate to read connection strings by setting the property **GetConnectionStringDelegate** of the **ConfigReader** class.   

**Note**: You only need to assign these delegates once when the application is starting up.   

#Getting Started
##Defining Item Types, Privileges and Permissions
Bodiam expects the user to define the security model based on the type of items to be secured (like an Order entity or a specific use case of the application), the privileges over those item types (like Read, Write, Grant, etc.) and finally the relationship between users or roles and these items as permissions.   

Bodiam already provides implementations of the User and the Role classes. Bodiam utilizes the concept of Role as a way to ease the management of a set of common permissions.   

The implementation of Privileges, Item Types, and Items is up to the user but must follow certain rules:   
- Privileges are defined as strings in code and stored in the database in a varchar(40) column   
- Item Types are defined as strings in code and stored in the database in a varchar(40) column   
- Items are stored as references to the actual item by its id but there is no referential integrity and currently is a Unique Global Identification (Guid)   

##Examples

Throughout this example, we will see all the steps needed to secure specific Items and Item Types. In addition, we will create a new User and a new Role.   
 
For these examples, you can create a blank Console Application or directly try the code snippets anywhere in your application. Always remember to reference the Bodiam assembly on the project you choose to work on.   

Frist add the following using statements for these examples   

```c#
    ...   
    using Bodiam.Entities;      
    using Bodiam.Services;   
    ...   
```

#Creating a new user
Bodiam provides an invitation mechanism to populate users but it also offers the IUserService to create users manually. In this case, we will use the CreateUser method.   
  
```c#
    var userService = new UserService();   
    var user = new User() { Name="Test User", Email = "test@email.com" };   
    userService.CreateUser(user);   
```

Note that the Name and Email fields are mandatory for the user creation. The CreateUser method will persist the new User in the Bodiam database.   

##Defining the Privileges and Items
Suppose we have two Item Types, “Products” and “Orders”. In addition, we want to have a “Check Price” and “Buy” permissions over products an “Emit” and “Pick Up” permits over orders.
To represent these Item Types and Privileges, we could create the following constants.
   
```c#
    internal static class Permissions 
    {   
        internal const string Products = "Products";  
        internal const string ProductsCheckPrice = "CheckPrice";   
        internal const string ProductsBuy = "Buy";   
        internal const string Orders = "Orders";   
        internal const string OrdersEmit = "Emit";   
        internal const string OrdersPickUp = "PickUp";   
    }      
```

Moreover, we can use nested classes to have item types and its privileges more organized.

```c#
    namespace Permissions
    {
        internal static class Products
        {
            internal const string Name = "Products";

            internal static class Privileges
            {
                internal const string CheckPrice = "CheckPrice";
                internal const string Buy = "Buy";
            }
        }
    
	    internal static class Orders
	    {
	            internal const string Name = "Orders";
	
	            internal static class Privileges
	            {
	               internal const string Emit = "Emit";
	                internal const string PickUp = "PickUp";
	            }
	        }
	    }   
	}
```

##Granting Privileges over an Item Type
  
Bodiam provides the IPermissionService which enables the user to grant (or revoke) privileges over Items.   

In the following code snippet, we will assign the "CheckPrice" permission to the "Test Customer" user for the "Product" Item type.   

```c#
    var userService = new UserService();
    var user = userService.RetrieveUserByName("Test User");
  
    var permissionService = new PermissionService();
  
    permissionService.AddPermissionToUser(user, Permissions.Products.Name, null,
      Permissions.Products.Privileges.CheckPrice);`   
```

**Note**: Bodiam handles the Item Id as Guid. Remember to keep track of such identifying property on your object model.   

##Enforcing Authorization
###Resource Based Authorization

For Resource Based Authorization you can use the [ResourceAuthorizeAttribute] class to decorate your classes and/or controllers. With this attribute, you can have Coarse-Grained or Fine-Grained authorization. We could also use the IPermissionService API to accomplish the same tasks.   

###Coarse grain Authorization

Coarse-grain authorization permits the checking of a privilege over an item group.   
 
  C#   
    `public class ProductController
    {
      ...
      [HttpGet]
      [ResourceAuthorize  (Privilege=Permissions.Products.Privileges.CheckPrice,                       
    ItemType=Permissions.Products.Name)]
        public ActionResult List()
        {
        ... ` 
 
This example uses Permissions.Products.Privileges.CheckPrice and Permissions.Products.Name constants. Notice that these two example constants are part of the application model and not from the Bodiam object model.   
 
Bodiam will check that the User or any of the User’s roles has the requested Privilege over the indicated Item Type.   

**Note**: Bodiam will only work with and persist the Privilege Id and the Item Type Id to abstract itself from their real representation. 

###Fine grain Authorization
Fine grain authorization permits the user to establish permission over a precise item instance. The Fine grain authorization usage is implicit. Every time the **ResourceAuthorizeAttribute** executes, it checks for the Id parameter in the Request Url. If the parameter is present, the authorization check will be performed over the Privilege Id, Item Type Id and Item Id.   

The **ResourceAuthorizeAttribute** will look for the Item Id as a RouteData value with key “Id”.   
 
###Using the IPermissionService API

The **ResourceAuthorizeAttribute** class relies in the **IPermissionService** for checking the user’s permission. We can explicitly use this service to accomplish the same level of authorization, both coarse and fine-grained.   

We can also use this service in the UI to display or hide elements according to the user’s permission.
Consider the following UI usage:   

  C# (cshtml)   
     <code>@using Bodiam.Services;
     @{
       var permissionService = new PermissionService();
       var userService = new UserService();
       var user = userService.RetrieveUserByName(
                  HttpContext.Current.User.Identity.Name);
        bool hasCheckPricePermission = false;   
    if (user != null)
       {
           var userPermissions = permissionService.GetPermissionsOnItemByUser(
            user, Permissions.Products.Name, Guid.Empty);
          hasCheckPricePermission = userPermissions.Any(
           p => p.Privilege == Permissions.Products.Privileges.CheckPrice);
       }
     }
     <p>
        Only users with Check Prices can see the Product price
     </p>
 
    @if(hasCheckPricePermission)
    {
         <div>
            Product Price: 50$
         </div>
    }
    else
    {
        <div>
            You do not have the proper permissions.
        </div>    
    }</code>   

In the former example, we are asking the IPermissionService if the current user has the “CheckPrice” privilege, and according to the response, we show or hide the price information.
Notice that we could have also passed to the IPermissionService a Guid for a specific item (instead of Guid.Empty), forcing fine-grained authorization. 

##Handling Unauthorized Access

If the user authorization fails, Bodiam will always redirect users to ~/Error/AccessDenied. Consider handling this the other defined routes in the Global.asax of simply implementing an ErrorController with an AccessDenied action.   
 
###Exceptions
Bodiam defines a custom CustomSecurityException. Bodiam will throw this exception in the context of invitation code validation.   
 
Here is a list of error codes and their error messages:   

- ERR001: The invitation ID does not exist. Please contact your administrator.   
- ERR002: The invitation was already activated.   
- ERR003: Your invitation has expired. Please contact your administrator.   
- ERR004: You are not the invited user   

##Invitations
Bodiam offers the possibility of sending Invitation Codes to outside users. Invitation Codes grant the users the chance to logon the application and become active users. Aside from the Invitations, users may explicitly request access to the application. Bodiam provides the means for keeping track of access requests and responding them.   

###Working with Access Requests
**Saving New Requests**   

Suppose we get back the users contact information for a Form. We will want to keep track of this information to consider inviting the user in the future.   

  C#
    var invitationService = new InvitationService();

    invitationService.SaveAccessRequest(new AccessRequest()
    {
       AccessRequestId = Guid.NewGuid(),
       Name = model.Name,
       Email = model.Email,
       Comment = model.Comment,
       CreateDateTime = DateTime.UtcNow
    });   

**Consuming Requests**   

To get the currentyou can simply consult the InvitationService   

  C#   
    var invitationService = new InvitationService();   
    IEnumerable<AccessRequest> requests = invitationService.GetRequests();   

**Inviting an User**   

For inviting new users, you will need to add the user to the Bodiam context via the UserService. Once this is done, use the InvitationService to send the invitation.   
 
An Invitation will need the following information to be correctly delivered:   
   
-  User (with a valid email address   
-  Invitation Link: Link that will be sent to the user with the invitation code. This link must point to your application. Remember to match this link with the UserAccountInvitationAction value defined in the application’s Web.Config.   
-  Expiration Date: A date in the future up to which the invitation is valid.   
-  Personal Message: A message written to the recipient of the invitation.   

**Note**: By default, the format of invitation e-mails is based on a template included in Bodiam as an embedded resource. If you want, you can provide a custom template by assigning a delegate in the **GetInvitationTemplateDelegate** property of the **ConfigReader** class.   

Optionally, you can add an Access Request Id for tracking the invitations back to the Access Request. You can also indicate the InvitationService to sign the invitation with the sender’s username.   

  C#   
    var userService = new UserService();
    var invitationService = new InvitationService();

    User user = new User() { Name = “User Name”, Email = “user@mail.com” };
   
    userService.CreateUser(user);
 
    var invitationLink = “Invitation/Accept”;
    var personalMessage = “Please accept this invitation.”
    var useSignature = false;
    Guid accessRequestId = null;

    var expiration = DateTime.UtcNow.AddDays(30);
 
    var invitation = invitationService.InviteUser(
               user, invitationLink, expiration, 
               personalMessage, useSignature, accessRequestId
               );   

**Accepting Invitations**   

You do not have to do anything special to accept users coming to the application with Invitation Codes. The CustomClaimsAuthenticationManager provided by Bodiam will check if the user has valid credentials. Being this the case, Bodiam will update the user information with that sent by the Identity Provider such as the Identity Provider Name.   

##Working with Roles
###Creating a new Role

Suppose we want to create a new role called “Customer”. Bodiam offers the IRoleService for role handling.   

  C#   
    var roleService = new RoleService();
    var role = new Role(){Name=’Customer’, Description=‘This is a sample role’};
    roleService.CreateRole(user);   


Just like with the UserService, the CreateRole method will insert the new role in the Bodiam database.   

###Assigning a Role to an User

Role managing is done in the IRoleService. This is also true for assigning roles to user or removing them from roles.   

We will use the RoleService to get the “Customer” role and the UserService to get the user named “Test User”. Finally, we will assign the “Customer” role to the “Test User”.   

  C#   
    var roleService = new RoleService();
    var userService = new UserService();

    var user = userService. RetrieveUserByName(‘Test User’);
    var role = roleService.GetRoleByName(‘Customer’);

    roleService.AssignRoleToUser(role, user);   

The AssignRoleToUser method will save the new relationship in the Bodiam database.   

Granting Privileges over an Item Type
Suppose we want to assign permission to the “Customer” role instead of to a particular user, so we can have the “CheckPrice” right over all our defined customers.
  C#   
    `var roleService = new roleService();
     var role = roleService.GetRoleByName(‘Customer’);
  
     var permissionService = new PermissionService();
  
     permissionService.AddPermissionToRole(role, Permissions.Products.Name,
      null, Permissions.Products.Privileges.CheckPrice);`   


###Role Based authorization

**Note**: Role Based Authorization is discouraged. Use Resource Based Authorization whenever possible.   

For Role Based authorization, you can use the [RoleAuthorizeAttribute] class to decorate your action or controllers.   
 
Roles can be identified by their name. Bodiam includes a role enumeration called SecurityRoles, which contains the following defined roles:   

-  SecurityAdmin   
-  SecurityUser   
-  HelpAdmin   

Feel free to use these or to create your own.   

  C#
    `[RoleAuthorize(Roles = SecurityRoles.SecurityAdmins)]
    public class HomeController
    {
      ...`   

Multiple roles can be entered as a single string with comma-separated values   

  C#
    `[RoleAuthorize(Roles = “User, PowerUser”)]
    public class HomeController
    {
      ... `   

#The Administration Interface
