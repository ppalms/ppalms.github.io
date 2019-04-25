---
layout: post
title: "ASP.NET Core Identity and Angular 2+ (Part 1)"
date: 2019-03-11
categories: programming bible-blast
---
## User Management
One of the basic requirements of the Bible Blast app is the ability for different users to log in to perform certain actions based on their role. There are some third party libraries like [Auth0](https://auth0.com) that do a good job at this, but I wanted to take the opportunity to learn about the [Identity framework](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-2.2&tabs=visual-studio) offered in .NET.

## .NET Core Identity Setup
By default, the framework will use the `IdentityUser` class (which maps to the `AspNetUsers` table). This entity contains basic information you would want on a user (username, email address, etc.). It's also possible to extend this class if you want your user to have additional fields. Entity Framework will automatically add these columns to the `AspNetUsers` table. In my case, I wanted to add a handful of fields and some navigation properties.

```csharp
public class User : IdentityUser<int>
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public int? OrganizationId { get; set; }
    public Organization Organization { get; set; }
    public ICollection<UserRole> UserRoles { get; set; }
    public ICollection<UserKid> Kids { get; set; }
    public bool IsActive { get; set; } = true;
}
```

I also extended the `IdentityRole` and `IdentityUserRole` classes (these map to the `AspNetRoles` and `AspNetUserRoles` tables respectively). `IdentityRole` represents a role in the system like 'Admin', 'Coach', or 'Member'. `IdentityUserRole` links user and role entities which have a many-to-many relationship.

I named my implementation of `IdentityUserRole` as `UserRole` and gave it navigation properties to the `User` entity as well as my `Role` entity (the base class only contains IDs for the user and role). My `Role` entity similarly contains a navigation property to its `UserRole` collection.

```csharp
public class UserRole : IdentityUserRole<int>
{
    public User User { get; set; }
    public Role Role { get; set; }
}

public class Role : IdentityRole<int>
{
    public ICollection<UserRole> UserRoles { get; set; }
}
```

These entities will ultimately be used to configure up the identity system during application startup.

### Startup.cs
The identity system gets configured in the `ConfigureServices()` method in the Startup class. To set up the identity system for your user type, you call the `AddIdentityCore<TUser>` extension method on `IServiceCollection`. This method also allows you to set up password requirements for your users. My dev environment setup roughly ended up looking like this:

```csharp
IdentityBuilder identityBuilder = services.AddIdentityCore<User>(opt =>
{
    opt.Password.RequiredLength = 8;
    opt.Password.RequireNonAlphanumeric = false;
    opt.Password.RequireDigit = false;
    opt.Password.RequireUppercase = false;
});

identityBuilder.AddRoles<Role>();
identityBuilder.AddEntityFrameworkStores<SqlServerAppContext>();
identityBuilder.AddRoleValidator<RoleValidator<Role>>();
identityBuilder.AddRoleManager<RoleManager<Role>>();
identityBuilder.AddSignInManager<SignInManager<User>>();
identityBuilder.AddDefaultTokenProviders();

Microsoft.IdentityModel.Logging.IdentityModelEventSource.ShowPII = true;
```
The last line tells .NET to go ahead and log sensitive user information rather than obfuscating it - you would not want this in your production setup.

This class is also responsible for setting up authentication. For my app, I'm using [JWT](https://jwt.io) token-based authentication. This section tells .NET how to build and verify the tokens that are used when a client makes a call to the backend of the application.

```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.ASCII.GetBytes(Environment.GetEnvironmentVariable("SHARED_KEY"))
        ),
        ValidateIssuer = false,
        ValidateAudience = false,
    });
```

(See the rest [here](https://github.com/ppalms/bible-blast/blob/master/BibleBlast.API/Startup.cs).)

### SqlServerAppContext.cs
The EF data context also needed to be modified so it would know how to build out the Identity-related tables. The overrides make this look a little gross, but the framework ultimately handles the heavy lifting.

```csharp
public partial class SqlServerAppContext : IdentityDbContext<User, Role, int,
        IdentityUserClaim<int>, UserRole, IdentityUserLogin<int>,
        IdentityRoleClaim<int>, IdentityUserToken<int>>
    {
        ...
    }
```

I opted to use integers as the data type for the identity columns out of personal preference (the framework uses GUIDs by default). I'd rather be dealing with endpoints like `/users/19` than `/users/b90fc973-73ff-4a06-8a49-cfdbae79489d`.

At this point, Entity Framework will create all of the necessary tables when the application is run, including additional columns for my extra properties in the extended `User` class.

### Web API Controllers and Authorization
With the identity system configured, the app can be secured based on whether a user is logged in as well as which role(s) they have. To lock down a controller entirely if a user is not logged in, all that's required is adding the `Authorize` attribute:
```csharp
namespace BibleBlast.API.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    [Authorize]
    public class KidsController : ControllerBase
    {
        ...
    }
}
```

This attribute also accepts a `Roles` argument for more fine-grained control, e.g. `[Authorize(Roles = "Coach,Admin")]`. An endpoint decorated this way would return a 401 Unauthorized response for users without the Coach or Admin role.

### Global query filters
Entity Framework has the concept of a global query filter which I used to filter data based on a user's organization. This way a user will not see data from other church groups that also use the application. Global query filters are set up in the `OnModelCreating()` method in the EF data context. The following example shows how you could exclude inactive Kid records:

```csharp
modelBuilder.Entity<Kid>().HasQueryFilter(x => x.IsActive);
```

While these filters will always be active by default, it's easy to get around them when you need to. Using the example above, I wouldn't want to restrict the data context in this way for administrators, and it can be disabled with the LINQ method `IgnoreQueryFilters()`:

```csharp
var kids = _context.Kids.AsQueryable();

if (userIsAdmin)
{
    kids = kids.IgnoreQueryFilters();
}

var kidList = await kids
    .Include(k => k.Parents)
    .ThenInclude(p => p.User)
    .ThenInclude(p => p.Organization)
    .ToListAsync();

return kidList;
```

At this point, the .NET side of things is up and running. It supports logging in and accepts JWT bearer tokens. After verifying that everything was working as expected in Postman, I moved on to the client side to set up Angular components to allow a user to login, and store and pass around the JWT token as needed. I'll cover that in Part 2.
