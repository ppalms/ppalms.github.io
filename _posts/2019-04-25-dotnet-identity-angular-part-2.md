---
layout: post
title: "ASP.NET Core Identity and Angular 2+ (Part 2)"
date: 2019-04-25
updated: 2019-05-10
categories: programming bible-blast
summary: The second in a series of posts covering implementation of the ASP.NET Identity system. This post looks at the frontend Angular components.
---
## Angular Components
In the [previous post]({% post_url 2019-03-11-dotnet-identity-angular-part-1 %}), I went over the backend implementation for user login/authentication. This post will cover the frontend components.

## Login form
Since this app is used to administer a closed course/curriculum, I didn't make a user registration page. Elevated users such as coaches and administrators will be responsible for creating user accounts. Instead, I started by embedding a login form in the navbar.

```
<form #loginForm="ngForm" *ngIf="!isLoggedIn()" class="form-inline" (ngSubmit)="login()">
    <input class="form-control" type="text" name="username" placeholder="Username" required
    [(ngModel)]="loginModel.username">
    <input class="form-control" type="password" name="password" placeholder="Password" required
    [(ngModel)]="loginModel.password">
    <button class="btn btn-success" type="submit" [disabled]="!loginForm.valid">Login</button>
</form>
```

I've removed most of the styling, but this is the gist of the login form. It's a simple form with only a couple fields, so it's a good place to start covering some Angular modules and conventions.

Here is an outline of the backing class:
```typescript
@Component({
  selector: 'app-nav',
  templateUrl: './nav.component.html',
  styleUrls: ['./nav.component.scss']
})
export class NavComponent {
  // The object that's bound to the form
  loginModel: any = {};

  constructor(public authService: AuthService, private router: Router) { }

  login() {
    // Authenticates the user and stores token 
    // and user info
  }

  logout() {
    // Clears token and user info
  }

  isLoggedIn() {
    // Returns a boolean indicating whether the 
    // current user is logged in
  }
}
```

The form makes use of Angular's template-driven [forms module](https://angular.io/guide/forms). 'Template-driven' refers to the fact that the form is defined in the HTML template. When this module is imported, Angular will automatically generate the necessary directives and handle data binding and validation for the `form` element(s) in the template.

Angular also has a `ReactiveFormsModule` that provides the ability to create model-driven forms. This basically approaches the problem from the other end, allowing you to define your form in the component class (the `*.ts` file), but it's a little more cumbersome and generally not used for simple scenarios.

### [(ngModel)]
The `ngModel` directive binds a control to its backing field. This sets up two-way data binding, meaning the model will register changes that the user makes from the UI and the UI will reflect any changes that the backing component class makes to the model.

The only validation on this form is that both fields are required. This is a standard HTML5 attribute, but Angular recognizes it and the form will be flagged as invalid if both fields aren't populated.

### #loginForm
The hash sign in front of `loginForm` indicates that this is a 'template variable', and the value that's being assigned refers to the form directive itself. This looks a little strange since `ngForm` is not declared anywhere, but the [framework guarantees](https://angular.io/api/forms/NgForm#description) that the form will have the `ngForm` directive applied to it. This variable provides a way to reference the form and its contents - in this case, it's being used to watch the `valid` property to determine whether to enable the Login button.

### *ngIf
The `*ngIf` directive is a ['structural' directive](https://angular.io/guide/structural-directives#what-are-structural-directives) (signified by the leading asterisk). Structural directives alter the DOM's structure in some way, and `*ngIf` will conditionally show or hide the element it's applied to.

In this case, it's being used to show or hide the login form and navigation buttons. The nav buttons are hidden and the login form is displayed if the user is not logged in:

![not logged in](../assets/images/not-logged-in.png "Not logged in")

And vice versa if the user is logged in:

![logged in](../assets/images/logged-in.png "Logged in")

### [disabled]
The `disabled` attribute makes use of the `loginForm` template variable to check whether the form passes validation. This is also an HTML5 directive, but notice it's enclosed in square brackets. The brackets tell Angular that the value being assigned needs to be evaluated rather than treated as a simple string value.

## Protected Routes
In Angular, a 'route' is a URL that points to a component. E.g., if you navigate to `/kids`, you'll be sent to a component called `KidListComponent`. This view should only be accessible to authenticated users. Although the navigation buttons would be hidden in this case, there's nothing to stop someone from typing the URL into the browser.

To address this issue, the Route object in Angular has a property called `canActivate` which can be used to allow/disallow user access to the component. `canActivate` expects an array of objects that implement the [`CanActivate`](https://angular.io/api/router/CanActivate) interface, and this interface has a single method called `canActivate()` which can return a boolean or a `UrlTree`. If the latter is returned, navigation is automatically canceled and the user is redirected to the specified URL.

Here is a portion of the [route configuration](https://angular.io/api/router/Route) with some comments for clarification:
```typescript
// When determining where to send a user, the framework  
// will evaluate these paths from the top down:
const routes: Routes = [
  // An empty path will always be considered a match.
  // '/' will send the user to the Home component
  { path: '', component: HomeComponent },
  // Any other path, e.g. /kids, will continue to look for 
  // a match, and the next route will again always match 
  // because it's an empty string. This is basically a 
  // cosmetic decision. All of the subsequent child routes 
  // could be lifted up to this level, but you would end up 
  // with a lot of repeated code for the 'runGuardsAndResolvers' 
  // and 'canActivate' properties.
  {
    path: '',
    runGuardsAndResolvers: 'always',
    canActivate: [AuthGuard],
    children: [
      { path: 'kids', component: KidListComponent, resolve: { kids: KidListResolver } },
      { path: 'kids/:id', component: KidDetailComponent, resolve: { kid: KidDetailResolver } },
      { path: 'users', component: UserListComponent, resolve: { users: UserListResolver } },
    ],
  },
  // The ** path is a wildcard, so if a user enters an 
  // invalid URL that isn't handled above, they will be 
  // redirected to '/'
  { path: '**', redirectTo: '', pathMatch: 'full' },
];
```

The `AuthGuard` class implements the `CanActivate` interface mentioned above, and the `canActivate()` method checks whether the user is logged in. The [final version](https://github.com/ppalms/bible-blast/blob/master/BibleBlast.SPA/src/app/_guards/auth-guard.ts) also checks which role a user is in.
```typescript
export class AuthGuard implements CanActivate {
    // I'll dig into AuthService later in this post
    constructor(private auth: AuthService, private router: Router) { }

    canActivate(next: ActivatedRouteSnapshot): boolean | UrlTree {
        if (this.auth.isLoggedIn()) {
            return true;
        }

        return this.router.parseUrl('home');
    }
}
```

The final step for setting up routing is to import `RouterModule` and set up the routes outlined above by calling `RouterModule.forRoot(routes)`.
```typescript
@NgModule({
  declarations: [
    AppComponent,
    NavComponent,
    HomeComponent,
    ...
  ],
  imports: [
    BrowserModule,
    RouterModule.forRoot(routes)
    ...
  ],
  providers: [
    KidListResolver,
    ...
  ],
  bootstrap: [
    AppComponent,
  ]
})
export class AppModule { }
```

Without getting too far into the weeds, `AppModule` is instantiated when the app runs and contains the various components and services that a [module](https://angular.io/guide/architecture-modules) needs in order to function. For more complicated applications, you may want to break functionality out into multiple modules. In this case, everything lives in the the foundational `AppModule`.

## AuthService
The `AuthService` class is responsible for communicating with the API and handling the [JWT token](https://jwt.io) and user information it receives.

The meat of the authentication service is the `login()` method. It sends a POST request to the `/api/login` endpoint. The body of the request contains a JSON object with the username and password of the user attempting to log in. The API returns a `401 Unauthorized` response if the credentials are invalid, or a `200 OK` response whose body contains an access token and user info.

Example response:
```
{
    "token": "eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJuYW...",
    "user": {
        "id": 19,
        "username": "tveal",
        "firstName": "Terry",
        "lastName": "Veal",
        "email": "terry.veal@AndAsItIsSuchSoAlsoAsSuchIsItUntoYou.com",
        "phoneNumber": null,
        "organization": {
            "id": 1,
            "name": "Church of the Good Shepherd"
        },
        "userRole": "Coach",
        "kids": []
    }
}
```

The `token` string will be held in the browser's local storage so it can be attached to subsequent API calls. The `user` info will be used by different components for display purposes (e.g., the nav bar will display a welcome message with the user's first name when they log in).

The JWT token contains the claims that were set up in [`AuthController.cs`](https://github.com/ppalms/bible-blast/blob/master/BibleBlast.API/Controllers/AuthController.cs):
```csharp
private async Task<string> GenerateJwtToken(User user)
{
    var claims = new List<Claim>
    {
        // The user ID will be stored in the "nameid" claim
        new Claim(ClaimTypes.NameIdentifier,user.Id.ToString()),
        // The username will be stored in the "unique_name" claim
        new Claim(ClaimTypes.Name, user.UserName),
        new Claim("organizationId", user.Organization?.Id.ToString() ?? string.Empty),
    };

    var roles = await _userManager.GetRolesAsync(user);
    foreach (var role in roles)
    {
        // The app also needs to know what role(s) the user has
        claims.Add(new Claim(ClaimTypes.Role, role));
    }
    
    ...
}
```

The server encodes these claims and spits out a big ugly string like this one:
```
eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJuYW1laWQiOiIzMCIsInVuaXF1ZV9uYW1lIjoidHZlYWwiLCJvcmdhbml6YXRpb25JZCI6IjIiLCJyb2xlIjoiQ29hY2giLCJuYmYiOjE1NTc0MjM1NDUsImV4cCI6MTU1NzUwOTk0NSwiaWF0IjoxNTU3NDIzNTQ1fQ.5wlzOLL8BSrX0o0HjqmM8lxg7xrOxsQcq6Qq7UfZqr_xv5rU-ebVjPT9yKNRr8NHP4NldwZp4cQdS8xbrxub4A
```

This string is what must be passed to the server in the header of subsequent HTTP requests made by the user. The client can also decode the token to access the claims it contains. The decoded token will be an object with properties for each of the claims associated with the user:
```
{
  "nameid": "30",
  "unique_name": "tveal",
  "organizationId": "2",
  "role": "Coach",
  "nbf": 1557423545,
  "exp": 1557509945,
  "iat": 1557423545
}
```

For more information, you can visit [JWT.io](https://jwt.io/#debugger) and paste the un-decoded token into the debugger.

### AuthService.login()
The `login()` method sends a POST request to the `/api/login` endpoint and stores the token and user info in local storage where the app can access it as needed. It also decodes the JWT token and stores the claims and user info on the `AuthService` instance.
```typescript
login(user: User) {
    return this.http.post(`${this.baseUrl}/login`, user)
      .pipe(map((response: any) => {
        if (response) {
          localStorage.setItem('token', response.token);
          localStorage.setItem('user', JSON.stringify(response.user));
          this.decodedToken = this.jwtHelper.decodeToken(response.token);
          this.currentUser = response.user;
        }
      }));
  }
```

`jwtHelper` is an instance of `JwtHelperService`, a class from the [@auth0/angular-jwt](https://www.npmjs.com/package/@auth0/angular-jwt?activeTab=readme) library. Auth0 is a third party authentication platform that I'm not using for this project since I wanted to learn how to implement .NET Identity. However, this library has some useful utilities that I didn't want to try to reinvent. For example, it can be set up to automatically include the user's access token in the header of every HTTP request sent to the server.

Setting up the Auth0 module required additional modifications to the `AppModule` class.

```typescript
// This function tells JwtModule how to get the token it will
// attach to HTTP requests
export function tokenGetter() {
  return localStorage.getItem('token');
}

@NgModule({
  imports: [
    ...
    JwtModule.forRoot({
      config: {
        tokenGetter,
        // Attach the token when making requests to these domains
        whitelistedDomains: ['localhost:5000'],
        // Do not attach the token when making requests to these routes
        blacklistedRoutes: ['localhost:5000/api/auth']
      }
    }),
    ...
  ],
  ...
})
export class AppModule { }
```

Finally, the `AppComponent` class needs to be updated to set the `decodedToken` and `currentUser` properties on `AuthService` when the component has been initialized. `AppComponent` is the base level component for the application; all of the other components/views essentially live inside of it.

The reason this code needs to exist is that `AuthService` is a singleton. When it is injected into the various components that depend on it, they assume the user data will be there. However, if a user refreshes their browser window, all application state is lost.

Setting these fields after `AppCompenent` initializes will ensure that the user won't encounter weird errors or be forced to log in again after a browser refresh.
```typescript
ngOnInit() {
  const token = localStorage.getItem('token');
  const user: User = JSON.parse(localStorage.getItem('user'));

  if (token) {
    this.authService.decodedToken = this.jwtHelper.decodeToken(token);
  }

  if (user) {
    this.authService.currentUser = user;
  }
}  
```

At this point, the Angular pieces of the puzzle are in place. The client app can access API endpoints that are decorated with the `[Authorize]` attribute on the user's behalf, and route guards are in place to keep unauthorized users from wandering around the client app without logging in.

## Conclusion
If it's not painfully obvious at this point, I'm still trying to find a balance between focusing on technical details and high level design. Is this a research paper or show-and-tell? It definitely seems like a little bit of both right now. I want to write about what I'm learning to help me solidify my understanding of new concepts and tools, so I know I've ended up focusing on certain points while completely glossing over others that I'm already somewhat familiar with.

My goal is to be helpful to other people and not just future me. Drop me a line at patrick [at] ppalmer.io if you'd like to see more information on a certain topic, or if you see something that's just plain wrong! The source code for this project is available on [Github](https://github.com/ppalms/bible-blast).