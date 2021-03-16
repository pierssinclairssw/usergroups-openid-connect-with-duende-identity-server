## OpenID Connect, OAuth2 with Duende Identity Server + React

This sample repo contains the minimum code to demonstrate OpenId Connect and Duende Identity Server. It contains the following projects:

- `ids-server`: Duende IdentityServer using In Memory provider
- `weatherapi`: API which is protected by the IdentityServer
- `react-client`: a React Client which allows user to login using Identity Server and communicate with the Weather API

## Step 1: Add new empty InMemory Duende IdentityServer

- create new Duende Identity Server project

```bash
mkdir ids-server
cd ids-server
dotnet new web
dotnet add package Duende.IdentityServer
```
- add `services.AddIdentityServer()` 

```csharp
// inside ConfigureServices() method
services.AddIdentityServer()
  .AddInMemoryClients(new List<Client>())
  .AddInMemoryIdentityResources(new List<IdentityResource>())
  .AddInMemoryApiResources(new List<ApiResource>())
  .AddInMemoryApiScopes(new List<ApiScope>())
  .AddTestUsers(new List<TestUser>());
```

- run `dotnet watch run` and open `https://localhost:5001/.well-known/openid-configuration` to see the open Id connect discovery doc
- compare it with https://accounts.google.com/.well-known/openid-configuration

## Step 2: Add new Web API project

- add new Web API project in the `weatherapi` folder

```bash
# on root folder
mkdir weatherapi
cd weatherapi
dotnet new webapi
```

- change `launchSettings.json` and update applicationUrl to `https://localhost:5002;http://localhost:5003`
- run `dotnet watch run` and open `https://localhost:5002/WeatherForecast` to see the json weather data 

## Step 3: protect weatherapi 

- add `services.AddAuthentication("Bearer")` and `app.UseAuthentication()` to startup.cs and `[Authorize]` on the `WeatherForecastController.cs`

```csharp

// inside ConfigureServices() method of weather API
services.AddAuthentication("Bearer")
  .AddJwtBearer("Bearer", options =>
  {
      options.Audience = "weatherapi";
      options.Authority = "https://localhost:5001";

      // ignore self-signed ssl 
      options.BackchannelHttpHandler = new HttpClientHandler { ServerCertificateCustomValidationCallback = delegate { return true; } };
  });


// inside Configure() method
app.UseRouting();

app.UseAuthentication();
app.UseAuthorization();


// WeatherForecastController.cs
[ApiController]
[Authorize]
[Route("[controller]")]
public class WeatherForecastController : ControllerBase 

```
- try open https://localhost:5002/WeatherForecast, you will get `401`

- add clients, resource and scope in the `startup.cs` of the IdentityServer

```csharp
// ConfigureServices(), startup.cs of ids-server
services.AddIdentityServer()
  .AddInMemoryApiScopes(new List<ApiScope> {
      new ApiScope("weatherapi.read", "Read Access to API"),
  })
  .AddInMemoryApiResources(new List<ApiResource>() {
      new ApiResource("weatherapi") {
          Scopes = { "weatherapi.read" },
      }
  })
  .AddInMemoryClients(new List<Client> {
      new Client
      {
          ClientId = "m2m.client",
          AllowedGrantTypes = GrantTypes.ClientCredentials,
          ClientSecrets = { new Secret("SuperSecretPassword".Sha256())},
          AllowedScopes = { "weatherapi.read" }
      }
  });
```

- get new access token by calling `POST https://localhost:5001/connect/token`

```curl
POST https://localhost:5001/connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&scope=weatherapi.read&client_id=m2m.client&client_secret=SuperSecretPassword
```
- use the token and call weatherapi (in Authorization header), you should now get data back

```curl
GET https://localhost:5002/WeatherForecast
Authorization: Bearer <TOKEN>
```
## Step 4: Add Quick start UI from Duende Quick start UI

- in ids-server folder
- run `curl -L https://raw.githubusercontent.com/DuendeSoftware/IdentityServer.Quickstart.UI/main/getmain.sh | bash` inside `ids-server`, this will download 3 new folders to the project: `QuickStart`,`Views` and `wwwroot`
- add `services.AddControllersWithViews();` 
- add `app.UseStaticFiles();` 
- add `app.UseEndpoints(endpoints => endpoints.MapDefaultControllerRoute());`

```csharp
// inside ConfigureServices() method
services.AddControllersWithViews();


// inside Configure() method
app.UseStaticFiles();
...
app.UseRouting();
app.UseIdentityServer();
app.UseEndpoints(endpoints => endpoints.MapDefaultControllerRoute());
```

- open https://localhost:5001/account/login and login using `alice` and `alice`

## Step 5: Add React Interactive Client

- in `ids-server` add test users `.AddTestUsers()`  and `AddInMemoryIdentityResources()`

```csharp
.AddInMemoryIdentityResources(new List<IdentityResource>() {
    new IdentityResources.OpenId(),
    new IdentityResources.Profile()
})
.AddTestUsers(new List<TestUser>() {
    new TestUser
        {
            SubjectId = "Alice",
            Username = "alice",
            Password = "alice"
        }
});
```

- add new interactive client `.AddInMemoryClients()` 

```csharp

.AddInMemoryClients(new List<Client> {
  // other clients.,
  new Client
  {
      ClientId = "interactive",

      AllowedGrantTypes = GrantTypes.Code,
      RequireClientSecret = false,
      RequirePkce = true,

      RedirectUris = { "http://localhost:3000/signin-oidc" },
      PostLogoutRedirectUris = { "http://localhost:3000" },

      AllowedScopes = { "openid", "profile", "weatherapi.read" }
  },
  })
```

- enable Cors on **both** ids_server and weather API by adding `services.AddCors();`, this will make sure React app can communicate with both backend servers

```csharp
// inside ConfigureServices() method for BOTH weatherapi and ids-server
services.AddCors();

// inside Configure() method  for BOTH weatherapi and ids-server
app.UseCors(config => config
    .AllowAnyOrigin()
    .AllowAnyHeader()
    .AllowAnyMethod()
);
app.UseRouting();
```

- create new React app by running `npx create-react-app react-client` 
- cd into the `react-client` and run `npm i oidc-client react-router-dom`
- replace content of `App()` function with the following

```jsx

// App.js
function App() {
  return (
    <BrowserRouter>
      <Switch>
        <Route path="/signin-oidc" component={Callback} />
        <Route path="/" component={HomePage} />
      </Switch>
    </BrowserRouter>
  );
}
```

- add OpenID connect config object

```javascript
const IDENTITY_CONFIG = {
  authority: "https://localhost:5001",
  client_id: "interactive",
  redirect_uri: "http://localhost:3000/signin-oidc",
  post_logout_redirect_uri: "http://localhost:3000",
  response_type: "code",
  scope: "openid weatherapi",
};
```

- in the same file add HomePage() component, this component will display `login` button if user is not logged in


```javascript
// App.js
function HomePage() {
  const [state, setState] = useState(null);
  var mgr = new UserManager(IDENTITY_CONFIG);

  useEffect(() => {
    mgr.getUser().then((user) => {
      if (user) {
        fetch("https://localhost:5002/weatherforecast", {
          headers: {
            Authorization: "Bearer " + user.access_token,
          },
        })
          .then((resp) => resp.json())
          .then((data) => setState({ user, data }));
      }
    });
  }, []);

  return (
    <div>
      {state ? (
        <>
          <h3>Welcome {state?.user?.profile?.sub}</h3>
          <pre>{JSON.stringify(state?.data, null, 2)}</pre>
          <button onClick={() => mgr.signoutRedirect()}>Log out</button>
        </>
      ) : (
        <>
          <h3>React Weather App</h3>
          <button onClick={() => mgr.signinRedirect()}>Login</button>
        </>
      )}
    </div>
  );
}
```

- in the same file add Redirect() component, this will handle OAuth2 redirect


```javascript
// App.js
function Callback() {
  useEffect(() => {
    var mgr = new UserManager({
      response_mode: "query",
    });

    mgr.signinRedirectCallback().then(() => (window.location.href = "/"));
  }, []);

  return <p>Loading...</p>;
}
```

- open http://localhost:3000, click on `Login` button
- login with `alice` and `alice`
- you should be redirected back to the React app with your name and weather information
- click `Logout`, you should be redirected back to the Identity Server, with a link to take you back to the React app