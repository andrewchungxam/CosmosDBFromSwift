# CosmosDBFromSwift
General Guidance around Accessing CosmosDB Securely From a (Swift) iOS App 

If you're developing a mobile app and looking through the CosmosDB Docs around security, you may notice that most of those documents seem to be referring to accessings CosmosDB from a web service.  You would be correct - most of those assume that you're accessing CosmosDB from a web service.

To translate to the mobile world - you might build something like this:
```
Mobile -> Middleware web app -> CosmosDB
```

Having the middleware as an intermediary is a recommended pattern in an enterprise app - it enhances control, enhances the customizability, adds various load-balancing functionality, etc. (You might consider Azure Functions, Azure App Service with a ASP.NET Core or ASP.NET Framework app, or an Azure API App).  And then if you want better monitoring, control, custom policies, visibility, and security...you can even add on top of that, a services like Azure API Management.

Secondly, one thing to note is that CosmosDB can define specific users and user permissions which are accessible via REST to access specific collections.  
Endpoints like this: <br />
https://docs.microsoft.com/en-us/rest/api/cosmos-db/create-a-user<br />
https://docs.microsoft.com/en-us/rest/api/cosmos-db/create-a-permission<br />

This is more accessibly described here: (the document is very information in nature but very clear)<br />
https://codemilltech.com/gimme-the-data-cosmos-db-permissions/  (The author is a Microsoft employee/ Azure+Xamarin evangelist)

You can see the official docs here:<br />
https://docs.microsoft.com/en-us/azure/cosmos-db/secure-access-to-data<br />
https://docs.microsoft.com/en-us/azure/cosmos-db/database-security<br />

You can also access CosmosDB with a MasterKey string - this MasterKey would be something you'd not want in your mobile client app, instead in your Azure App Service (ie the middeleware).  And as a next step, you can futher secure that, via KeyVault which has additional logging and security features.

One of the ideas you'll see recurring below is that you want to avoid coding confidential strings into your mobile apps -- in case malicious actors get access to your phone/app, decompile them, and having access to your confidential string.

---------

Now there are variations on the above ideas that generally allow you to proceed in the following ways.

Method A)
Mobile app --> Middleware web app -> CosmosDB

- Middleware app with the CosmosDB MasterKey gets data from CosmosDB and then passes it onto the mobile app
- You would be using something like Azure AD or Azure AD B2C to securely access the web service from your mobile device

Method B) 
Mobile app --> Middleware web app (Permission broker) -> CosmosDB

- Middleware acts as a permissions broker.  Mobile uses Azure AD to securely access the middleware and then the tokens are passed to the mobile app which then uses those tokens to access CosmosDB.

This is described here: <br />
https://codemilltech.com/connecting-to-cosmos-without-conx-strings/

---------

So where would these implementations be?

So in Method A - there are two SDKs that are available for you to use.

ADAL: https://github.com/AzureAD/azure-activedirectory-library-for-objc <br />
MSAL: https://github.com/AzureAD/microsoft-authentication-library-for-objc <br />

If you're using Azure AD - both SDKs are fine for you to use; note the MSAL still says "Preview", but you'll see on Github the following:

These libraries are suitable to use in a production environment. We provide the same production level support for these libraries as we do our current production libraries. During the preview we reserve the right to make changes to the API, cache format, and other mechanisms of this library without notice which you will be required to take along with bug fixes or feature improvements This may impact your application. For instance, a change to the cache format may impact your users, such as requiring them to sign in again and an API change may require you to update your code. When we provide our General Availability release later, we will require you to update your application to our General Availabilty version within six months to continue to get support.

Informally speaking, MSAL is where insiders will say, use this library; it's where there is a lot of energy/attention for future investment and importantly also you can use it for both Azure AD and Azure B2C where as ADAL cannot be used with Azure B2C.

Want to see examples? Take a look here: <br />
ADAL: https://azure.microsoft.com/en-us/resources/samples/active-directory-ios/ <br />
MSAL: https://github.com/Azure-Samples/active-directory-ios-swift-native-v2  <br />

Now - you'll be needing that Middleware.  If you're not sure which one to use - try using Azure Functions.

Searching for the terms: Functions and Azure AD will turn up a lot in terms of helpful documetns.

One of the clearest walkthroughs I found is with Azure AD B2C + Xamarin (even if you're not specifically using B2C or Xamarin - it might be a worth looking through as it does a pretty solid job walking through the fundamentals of the design): <br />
https://codemilltech.com/adding-azure-ad-b2c-authentication-to-azure-functions/

And also this example from one of the Xamarin evangelists: <br />
https://github.com/brminnick/XamList  (He uses different technology on the DB side - but you're looking at this to look at the Functions piece.)

And also the following example from the actual documentation: <br />
https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-cosmosdb-v2 <br />
https://docs.microsoft.com/en-us/azure/cosmos-db/sql-api-dotnet-samples (Users, Permissions, and Querying) <br />

----------

Method B)

The example for Method B - would be this document: <br />
https://codemilltech.com/connecting-to-cosmos-without-conx-strings/

---------

### Azure.Mobile
Templates and SDKs to help speed up your development.

AzureMobile is a project started by two Microsoft employees (@naterickard and @colbylwilliams): <br />
https://github.com/Azure/Azure.Mobile <br />
https://github.com/Azure/Azure.iOS <br />
https://github.com/Azure/Azure.Android <br />

This is an open source project on the official Azure github page but it's not an official "product" of Microsoft so there is not an SLA associate with it.  That said, it's all open source which you can of course look through; many parties including partners, customers, and internal teams are taking hard looks at it, and are either using it directly for their apps or using it for their inspiration for products they build off of it.

There are a lot of packages, and there are a variety of ways to use the packages.

One note - if you click around the project you might notice that there is a way to add the Master Key of your Function in Azure.Data (which would put the master string in your mobile app); however, this is for convenience when quickly developing samples and is not recommended to ship these values in the production apps.  (Again, the principle of not putting your master key in the mobile app still stands for production scenarios.)

Instead - use the Authentication provider of your choice.  And those Auth providers often will provide SDKs.  For example, you can ues either of the two Azure AD SDKs mentioned above (ADAL or MSAL).  Then AzureAuth of this project (Azure.iOS or Azure.Android) will work in tandem - you'll just need the access token that you will get back from the Authentication provider you used.

Your code will look something like this:

```
import AzureAuth
import AzureData
import AzureMobile

// Use MSAL SDK to login and get accessToken
let accessToken: String = ""
let functionsUrl = URL("FunctionsUrl")

AzureAuth.login(to: functionsUrl, with: .aad(accessToken: accessToken)) { response in 
    AzureData.configure(forAccountNamed: "documentDbName", withPermissionProvider: DefaultPermissionProvider(withBaseUrl: functionsUrl))
}
```

In the Azure.Mobile link you'll notice there is an ARM template which allows you to with one click from the Github portal, create various Azure services.  You'll see a Function spin up in a couple minute - you can use that Function to either go Method A or B.  NOTE - at this point, you need to use the C# version (Not Javascript, not C# Script to get all the additional code).  There will be code pre-loaded for the Function to be used as a permissions broker similar to what was described in Method B.  (Note - if your ARM template fails for some reason, try deploying it again with all the same values you did the first time and without deleting anything.)

Finally, there is a lot of code in the AzureData module that makes it convenient and easy to access and write data to your CosmosDB; there is also the ability to enable offline capabilities and cache data locally.  

---------

I hope the above is helpful.

Security is a big topic; enterprises should spend the appropriate time on this area of their projects and I often tell customers to work with a trusted consulting partner to do a full security audit on their projects.

 ---------
 
 Thanks to @naterickard and @colbylwilliams for their advice and super clear explainations

Edit: another great write up from a colleague of mine: https://github.com/adamhockemeyer/Azure-Functions---CosmosDB-ResourceToken-Broker

