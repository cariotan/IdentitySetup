# Not for Blazor Server due incompatibility of SignInManager which utilizes cookies.

### Authentication

The most minimum code needed to add basic authentication capabilities such as register and log in users. Use `SignInManager` and `UserManager` to add those capabilities.

Add the following nuget packages to your project:
```
Microsoft.EntityFrameworkCore
Microsoft.AspNetCore.Identity.EntityFrameworkCore
Microsoft.EntityFrameworkCore.SqlServer
Microsoft.EntityFrameworkCore.Design - this is required for ef tools to work.
```
Make sure the packages uses the same versions.

The model that represents the identity database is the `IdentityDbContext`. This is a pre-made context that comes from `Microsoft.AspNetCore.Identity.EntityFrameworkCore`.

Now you need to add `IdentityDbContext`, `SignInManager `and `UserManager` to services in `program.cs` so it can be used for dependency injection. Both `SignInManager` and `UserManager` is added using `AddIdentity`. Read below to setup `IdentityDbContext`.

#### IdentityDbContext

You cannot perform a database migration with `IdentityDbContext` as it currently resides in another assembly. So create a new class that inherits from `IdentityDbContext`. Next will depend on whether you are setting up the sql connection from services are in the class itself. If in the class itself, add this method:
```
	protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
	{
		optionsBuilder.UseSqlServer({connectionString});
	}
```
otherwise, 
```
	public IdentityPracticeContext(DbContextOptions<IdentityPracticeContext> options)
			: base(options)
		{

		}
```

To setup the string connection, add the below to program.cs

```
builder.Services.AddDbContext<IdentityDbContext>(options => options.UseSqlServer(builder.Configuration.GetConnectionString("IdentityPractice")));
```

In appsettings.json

```
  "ConnectionStrings": {
        "IdentityPractice": "server=.\\SQLEXPRESS;database=IdentityPractice;Trusted_Connection=true"
    }
```

Next, append `AddEntityFrameworkStores<ChildIdentityDbClassName>();`.

That is needed to register the necessary services that are required for authentication and authorization services.

After that, then you can perform the first migration to create the necessary tables in the sql database. Without `AddEntityFrameworkStores`, the migration wouldn't know what structure the authentication and authorization system will be as you can be using a custom structor or third party external providers.

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

Recommended by the top reply to not have the timestamp to be below 2 minutes.
Calling `UpdateSecurityStampAsync` isn't enough as even though the security stamp has been updated, it isn't validated on every request until the interval has passed.

### Authorization

Use `AuthorizeView` component to control the UI based on the authentication state. `AuthorizeView` obtains it from a cascading parameter: `CascadingAuthenticationState`. The state of the authentication is obtained from `AuthenticationStateProvider`.
The above components are in the nuget package: `Microsoft.AspNetCore.Components.Authorization`, so add that to your project. Add `AddAuthorizationCore` to services to inject required services to `AuthorizeView`. An implementation of `AuthenticationStateProvider` will need to be created and added to services.

#### `AuthenticationStateProvider`
As mentioned previously, the state of the authentication that `AuthorizeView` uses is provided by `AuthenticationStateProvider`.  A concrete implementation of that class will need to be provided. It contains a single method that needs implementation - `GetAuthenticationStateAsync`. The method return a `AuthenticationState` object that contains the `ClaimsPrincipal` that will be used to determine the state of the authentication. The primary logic of `GetAuthenticationStateAsync` will be to create the `ClaimsPrincipal` and return it inside a `AuthenticationState` object. `ClaimsPrincipal` are made up of one or more `ClaimsIdentity` which in turn is made up of one or more `Claim`. It would be considered autehnticated if `AuthenticationType` property of `ClaimsIdenity` is a non-null value and that claims exist that matches the existing criteria of `AuthorizeView`.

In addition, `AuthenticationStateChanged` event will need to be invokes whenever the state of authentication changes and the view needs to be updated immediately. To invoke it, `NotifyAuthenticationStateChanged` will need to be called. The method is a protected method so a public wrapper will need to be written and calling that instead to notify any changes in the authentication state.

Whem implementing `GetAuthenticationStateAsync()`, the set of claims can be obtained in the server HttpContext.User.Claims. The `ClaimIdentity` that you pass to the `ClaimsPrincipal` will require the `authenticationType` to be specified. 

#### `[Authorize]` attribute
Add the middlewares after app.Routing:
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

### Two-Factor Authentication

1. Create a class that implements the `IUserTwoFactorTokenProvider<IdentityUser>` interface. This class will provide the implementation for generating and validating 2FA tokens for users.

2. Add the token provider implementation to the identity builder service by using the `AddTokenProvider` method. Provide a name for the token provider during registration.

3. Once the token provider is registered, you can enable 2FA for a user by calling `userManager.SetTwoFactorEnabledAsync(user, true);`. This method will set the 2FA flag for the specified user.

4. When calling the `signInManager.PasswordSignInAsync` method, check the returned result to determine if 2FA is required for the user. If the token provider is not provided, this method will always return false.

5. To generate a 2FA code for the user, call `userManager.GenerateTwoFactorTokenAsync`. This will generate a code that the user can use for authentication.

6. If the user has entered the correct email and password, you can proceed to log them in by calling `signInManager.TwoFactorSignInAsync`. This method will validate the 2FA code provided by the user and sign them in if it's correct.
