

### Authentication

The most minimum code needed to add basic authentication capabilities such as register and log in users. Use `SignInManager` and `UserManager` to add those capabilities.

Add the following nuget packages to your project:
```
Microsoft.EntityFrameworkCore
Microsoft.AspNetCore.Identity.EntityFrameworkCore
Microsoft.EntityFrameworkCore.SqlServer
```
The model that represents the identity database is the `IdentityDbContext`. This is a pre-made context that comes from `Microsoft.AspNetCore.Identity.EntityFrameworkCore`.

Now you need to add `IdentityDbContext`, `SignInManager `and `UserManager` to services in `program.cs` so it can be used for dependency injection. `IdentityDbContext` is added using `AddDbContext` and both `SignInManager` and `UserManager` is added using `AddIdentity`.

#### Configuring the identity system
Example of password requirements configuration:
```
builder.Services.Configure<IdentityOptions>(options =>
{
	options.Password.RequireDigit = false;
	options.Password.RequireLowercase = false;
	options.Password.RequireNonAlphanumeric = false;
	options.Password.RequireUppercase = false;
	options.Password.RequiredLength = 6;
	options.Password.RequiredUniqueChars = 1;
});
```

#### Security stamp

The security stamp track changes made to the profile such as password changes. When you change a password, a new security stamp is generated. This would invalidate any cookies that has the old stamp. However, you may notice that users who are already signed in before the password change are still signed in. This is because security stamps are validated at a set interval. The default is 30 minutes. This can be changed via:

```
builder.Services.Configure<SecurityStampValidatorOptions>(options =>
{
	options.ValidationInterval = new(0, 2, 0);
});
```

https://stackoverflow.com/questions/39659876/asp-net-identity-reset-cookies-and-session-on-iis-recycle-restart

Recommended by the top reply to not have the timmestamp to be below 2 minutes.
Calling `UpdateSecurityStampAsync` isn't enough as even though the security stamp has been updated, it isn't validated on every request until the interval has passed.

### Authorization

Use `AuthorizeView` component to control the UI based on the authentication state. The state of the authentication is obtained from `AuthenticationStateProvider`. `AuthorizeView` obtains it from a cascading parameter: `CascadingAuthenticationState`.

The above components are in the nuget package: `Microsoft.AspNetCore.Components.Authorization`, so add that to your project. Add `AddAuthorizationCore` to services to inject required services to `AuthorizeView`. An implementation of `AuthenticationStateProvider` will need to be created and added to services.

#### `AuthenticationStateProvider`
As mentioned previously, the state of the authentication that `AuthorizeView` uses is provided by `AuthenticationStateProvider`.  A concrete implementation of that class will need to be provided. It contains a single method that needs implementation - `GetAuthenticationStateAsync`. The method return a `AuthenticationState` object that contains the `ClaimsPrincipal` that will be used to determine the state of the authentication. The primary logic of `GetAuthenticationStateAsync` will be to create the `ClaimsPrincipal` and return it inside a `AuthenticationState` object. `ClaimsPrincipal` are made up of one or more `ClaimsIdentity` which in turn is made up of one or more `Claim`. It would be considered au tehnticated if `AuthenticationType` property of `ClaimsIdenity` is a non-null value and that claims exist that matches the existing criteria of `AuthorizeView`. In addition, `AuthenticationStateChanged` event will need to be invokes whenever the state of authentication changes and the view needs to be updated immediately. To invoke it, `NotifyAuthenticationStateChanged` will need to be called. The method is a protected method so a public wrapped will need to be written and calling that instead to notify any changes in the authentication state.

#### `[Authorize]` attribute
Add the middlewares:
```
app.UseAuthentication();
app.UseAuthorization();
```
`UseAuthentication` is needed because authorize needs to know if the user is authenticated. `UseAuthorization` is self-explanatory.

### Cookie invalidation interval

To change when cookies are validated:

```

builder.Services.Configure<SecurityStampValidatorOptions>(options =>
{
	options.ValidationInterval = new(1);
});
```
