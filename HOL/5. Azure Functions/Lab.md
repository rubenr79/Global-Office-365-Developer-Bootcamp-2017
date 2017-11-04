# Creando una aplicación de consola usando Microsoft Graph
----------------
En este laboratorio crearemos de una aplicación de consola. NET desde cero utilizando. NET Framework 4.6.2, el Microsoft Graph SDK y Microsoft Authentication Library (MSAL).


## Prerequisitos

Este laboratorio usa Visual Studio 2017. También requiere un tenant con Azure AD y un usuario con privlegios de administrador.

### Registrando la aplicación

En un navegador web accede a [Application Registration Portal](https://apps.dev.microsoft.com/) para registrar la aplicación.

Haz click en el botón **Add an app**.

![](Images/01.png)

En la siguiente página introduce el nombre de la aplicación.

![](Images/02.png)

Haz click en el botón **Add Platform**. En la ventana que se muestra selecciona **Native Application**.
![](Images/03b.png)

Una vez se cree la aplicación se generará un Applicacion Id. **Copia este ID**, lo usaremos como el Client ID en el fichero `app.config` de la aplicación de consola.

![](Images/03.png)

Una vez que hayas terminado, asegurate de pulsar el botón **Guardar** para salvar los cambios.


![](Images/03f.png)

### Crea el proyecto en Visual Studio 2017

En Visual Studio 2017, crea un nuevo proyecto de tipo **Console Application** que utilice .NET Framework 4.6.2.

![](Images/04.png)

En el menú ve a Herramientas / Administrador de paquetes Nuget / **Consola de administración de paquetes**. En esta ventana ejecuta los siguientes comandos:

````powershell
Install-Package "Microsoft.Graph"
Install-Package "Microsoft.Identity.Client" -pre
Install-Package "System.Configuration.ConfigurationManager"
````

Edita el archivo  app.config, y antes del elemento  &lt;/configuration&gt;, añade el siguiente fragmento:

````xml
<appSettings>
    <add key="clientId" value="<application ID>"/>
</appSettings>
````

Reemplaza **<application ID>** con el valor del  **Application ID** que se generó en el portal de registro de aplicaciones.

### Añade AuthenticationHelper.cs

Añade una clase al proyecto que se llame **AuthenticationHelper.cs**. Esta clase será la responsable de la autenticación usando Microsoft Authentication Library (MSAL), que es el paquete **Microsoft.Identity.Client** que hemos instalado.


Reemplace la sentencia using en la parte superior del archivo por el siguiente fragmento.

````csharp
using Microsoft.Graph;
using Microsoft.Identity.Client;
using System;
using System.Configuration;
using System.Diagnostics;
using System.Linq;
using System.Net.Http.Headers;
using System.Threading.Tasks;
````

Reemplaza la declaración de la clase con lo siguiente:

````csharp
public class AuthenticationHelper
{
    // The Client ID is used by the application to uniquely identify itself to the v2.0 authentication endpoint.
    static string clientId = ConfigurationManager.AppSettings["clientId"].ToString();
    public static string[] Scopes = { "User.Read" , "User.ReadBasic.All"};

    public static PublicClientApplication IdentityClientApp = new PublicClientApplication(clientId);

    private static GraphServiceClient graphClient = null;

    public static GraphServiceClient GetAuthenticatedClient()
    {
        if (graphClient == null)
        {
            // Create Microsoft Graph client.
            try
            {
                graphClient = new GraphServiceClient(
                    "https://graph.microsoft.com/v1.0",
                    new DelegateAuthenticationProvider(
                        async (requestMessage) =>
                        {
                            var token = await GetTokenForUserAsync();
                            requestMessage.Headers.Authorization = new AuthenticationHeaderValue("bearer", token);
                        }));
                return graphClient;
            }

            catch (Exception ex)
            {
                Debug.WriteLine("Could not create a graph client: " + ex.Message);
            }
        }

        return graphClient;
    }


    public static async Task<string> GetTokenForUserAsync()
    {
        AuthenticationResult authResult = null;
        try
        {
            authResult = await IdentityClientApp.AcquireTokenSilentAsync(Scopes, IdentityClientApp.Users.FirstOrDefault());
            return authResult.AccessToken;
        }
        catch (MsalUiRequiredException ex)
        {
            // A MsalUiRequiredException happened on AcquireTokenSilentAsync. 
            //This indicates you need to call AcquireTokenAsync to acquire a token

            authResult = await IdentityClientApp.AcquireTokenAsync(Scopes);
            
            return authResult.AccessToken;
        }    
        
    }

    public static void SignOut()
    {
        foreach (var user in IdentityClientApp.Users)
        {
            IdentityClientApp.Remove(user);
        }
        graphClient = null;        

    }
}
````

### Obtener el perfil del usuario usuando Graph SDK

Microsoft Graph API facilita la consulta del perfil del usuario actualmente conectado. Esta muestra usa nuestra clase `AuthenticationHelper. cs` para obtener un cliente autenticado antes de acceder al endpoint **Me**. 


**Edita** la clase `Program.cs` y reemplaza el using generado con lo siguiente:

````csharp
using Microsoft.Graph;
using Newtonsoft.Json.Linq;
using System;
using System.Diagnostics;
using System.Net.Http;
using System.Threading.Tasks;
````

Para obtener la información de perfil del usuario actualmente conectado, añade el siguiente método: 

````csharp
public static async Task<User> GetMeAsync()
{
    User currentUserObject = null;
    try
    {
        var graphClient = AuthenticationHelper.GetAuthenticatedClient();
        currentUserObject = await graphClient.Me.Request().GetAsync();    
                        
        Debug.WriteLine("Got user: " + currentUserObject.DisplayName);
        return currentUserObject;
    }

    catch (ServiceException e)
    {
        Debug.WriteLine("We could not get the current user: " + e.Error.Message);
        return null;
    }            
}
````

### Obten los usuarios relacionados con el usuario usando REST API

Microsoft Graph API provee de endpoints REST para acceder a la información. Uno de estos endpoins es me/people que obtiene información sobre las personas conectadas con el usuario actual. Este método demuestra como se puede hacer llamadas a Microsoft Grap utilizando su API, para ello utilizaremos `System.Net.HttpClient` y añadiremos el token de acceso en la cabecera de Autorización.

````csharp
static async Task<string> GetPeopleNearMe()
{
    try
    {
        //Get the Graph client
        var graphClient = AuthenticationHelper.GetAuthenticatedClient();
        
        var token = await AuthenticationHelper.GetTokenForUserAsync();

        var request = new HttpRequestMessage(HttpMethod.Get, graphClient.BaseUrl + "/me/people");
        request.Headers.Authorization = new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);

        var response = await graphClient.HttpProvider.SendAsync(request);
        var bodyContents = await response.Content.ReadAsStringAsync();

        Debug.WriteLine(bodyContents);
        return bodyContents;
    }

    catch (Exception e)
    {
        Debug.WriteLine("Could not get people: " + e.Message);
        return null;
    }
}
````

### Todo junto

Copiaremos este método en Program.cs. Este método utiliza el patron async/await para obteneros los datos de las llamadas a Microsoft Graph.

````csharp
static async Task RunAsync()
{
    //Display information about the current user            
    Console.WriteLine("Get My Profile");
    Console.WriteLine();

    var me = await GetMeAsync();

    Console.WriteLine(me.DisplayName);
    Console.WriteLine("User:{0}\t\tEmail:{1}", me.DisplayName, me.Mail);
    Console.WriteLine();

    //Display information about people near me
    Console.WriteLine("Get People Near Me");

    var peopleJson = await GetPeopleNearMe();
    dynamic people = JObject.Parse(peopleJson);
    if(null != people)
    {
        foreach(var p in people.value)
        {
            var personType = p.personType;
            Console.WriteLine("Object:{0}\t\t\t\tClass:{1}\t\tSubclass:{2}", p.displayName, personType["class"], personType.subclass);
        }
    }
}
````
Finalmente, modifica el metodo Main de Program.cs para llamar al método RunAsync() que hemos creado.


````csharp
static void Main(string[] args)
{
    RunAsync().GetAwaiter().GetResult();
}
````

Run the application. You are prompted to log in.

![](Images/05.png)

Ejecuta la aplicación, la salida será similar a esto:

![](Images/06.png)


