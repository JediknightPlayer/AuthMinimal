# Google OAuth 2.0 Integration Guide

## Overview
This guide explains how Google OAuth 2.0 has been integrated into the GDE Edu AI Blazor application using the Authorization Code Flow.

## Architecture

### 1. OAuth Flow Implementation

The implementation follows the standard OAuth 2.0 Authorization Code Flow:

```
User → Click "Sign in with Google"
     → Redirect to Google OAuth consent screen
     → User grants permission
     → Google redirects back with authorization code
     → Backend exchanges code for tokens
     → Extract user claims from ID token
     → Create or authenticate user in database
     → Establish authenticated session
```

### 2. Key Components

#### Program.cs Configuration
```csharp
builder.Services
    .AddAuthentication(options =>
    {
        options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = GoogleDefaults.AuthenticationScheme;
    })
    .AddCookie(options =>
    {
        options.LoginPath = "/signin";
        options.ExpireTimeSpan = TimeSpan.FromDays(30);
        options.SlidingExpiration = true;
    })
    .AddGoogle(options =>
    {
        options.ClientId = configuration["Authentication:Google:ClientId"];
        options.ClientSecret = configuration["Authentication:Google:ClientSecret"];
        options.CallbackPath = "/signin-google";
        options.SaveTokens = true;

        // Request additional scopes
        options.Scope.Add("profile");
        options.Scope.Add("email");
    });
```

**Key Points:**
- `DefaultScheme`: Cookie-based authentication for maintaining session
- `DefaultChallengeScheme`: Google OAuth for external authentication
- `CallbackPath`: Where Google redirects after authentication
- `SaveTokens`: Stores access/refresh tokens for future API calls
- `Scopes`: Request profile and email information

#### Signin.razor (Login Button)
```csharp
private async Task SignInWithGoogle()
{
    var httpContext = httpContextAccessor.HttpContext;
    var properties = new AuthenticationProperties
    {
        RedirectUri = "/signin-google"
    };

    await httpContext.ChallengeAsync(
        GoogleDefaults.AuthenticationScheme,
        properties);
}
```

**What happens:**
1. `ChallengeAsync` triggers OAuth flow
2. User is redirected to Google's consent screen
3. After consent, Google redirects to `/signin-google`

#### GoogleCallback.razor (OAuth Callback Handler)
This page handles the OAuth callback and extracts user information:

```csharp
// Authenticate with Google
var authenticateResult = await httpContext.AuthenticateAsync(
    CookieAuthenticationDefaults.AuthenticationScheme);

// Extract claims from Google ID token
var principal = authenticateResult.Principal;
var email = principal?.FindFirst(ClaimTypes.Email)?.Value;
var name = principal?.FindFirst(ClaimTypes.Name)?.Value;
var givenName = principal?.FindFirst(ClaimTypes.GivenName)?.Value;
var surname = principal?.FindFirst(ClaimTypes.Surname)?.Value;
var nameIdentifier = principal?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
var picture = principal?.FindFirst("picture")?.Value;
```

**Available Claims from Google ID Token:**
- `ClaimTypes.Email`: User's email address
- `ClaimTypes.Name`: Full name
- `ClaimTypes.GivenName`: First name
- `ClaimTypes.Surname`: Last name
- `ClaimTypes.NameIdentifier`: Unique Google User ID
- `"picture"`: Profile picture URL
- `"email_verified"`: Whether email is verified

### 3. User Creation/Authentication Flow

```csharp
private async Task<LoginResultModel?> AuthenticateOrCreateUser(
    string email, string? givenName, string? surname,
    string? fullName, string? googleId, string? pictureUrl)
{
    // Try to login with Google ID marker
    var googleLoginModel = new LoginModel
    {
        Email = email,
        Password = $"GOOGLE_OAUTH_{googleId}"
    };

    var loginResult = await authService.Login(googleLoginModel);

    // If user doesn't exist, create new user
    if (loginResult?.Result?.Success == false)
    {
        var newUser = new UserModel
        {
            Email = email,
            FirstName = givenName ?? "Google",
            LastName = surname ?? "User",
            Password = $"GOOGLE_OAUTH_{googleId}", // Special marker
            Active = true,
            Guid = googleId,
            Roles = new List<RoleModel> { ... }
        };

        await userService.AddUser(newUser);
        loginResult = await authService.Login(googleLoginModel);
    }

    return loginResult;
}
```

**Important Notes:**
- OAuth users have a special password marker: `GOOGLE_OAUTH_{googleId}`
- This prevents traditional login conflicts
- User profile is auto-populated from Google data
- User is activated immediately (no email verification needed)

### 4. AuthenticationStateProvider Integration

The `CustomAuthentication` class tracks authentication state:

```csharp
public async Task MarkUserAsAuthenticated(LoginResultModel user)
{
    await _localStorage.SetItemAsync("token", user.Token);

    LoginTokenModel loginTokenModel = new LoginTokenModel { Token = user.Token };
    LoginUserModel loggedUser = await _authenticationService.GetUserFromToken(loginTokenModel);

    ClaimsIdentity identity = GetClaimsIdentity(loggedUser);
    ClaimsPrincipal claimsPrincipal = new ClaimsPrincipal(identity);

    NotifyAuthenticationStateChanged(
        Task.FromResult(new AuthenticationState(claimsPrincipal)));
}
```

**How Claims Work:**
1. After successful OAuth, backend generates JWT token
2. Token is stored in local storage
3. `GetClaimsIdentity` creates claims from user data
4. `NotifyAuthenticationStateChanged` updates auth state across app
5. All `[Authorize]` attributes now recognize the user

### 5. Accessing User Information in Components

```razor
@attribute [Authorize]

<AuthorizeView>
    <Authorized>
        <p>Welcome, @context.User.Identity.Name!</p>
        <p>Email: @context.User.FindFirst(ClaimTypes.Email)?.Value</p>
        <img src="@profilePicture" alt="Profile" />
    </Authorized>
    <NotAuthorized>
        <p>Please log in</p>
    </NotAuthorized>
</AuthorizeView>

@code {
    [CascadingParameter]
    private Task<AuthenticationState> authenticationState { get; set; }

    private string profilePicture;

    protected override async Task OnInitializedAsync()
    {
        var authState = await authenticationState;
        var user = authState.User;

        if (user.Identity.IsAuthenticated)
        {
            var email = user.FindFirst(ClaimTypes.Email)?.Value;
            var name = user.FindFirst(ClaimTypes.Name)?.Value;
        }
    }
}
```

## Setup Instructions

### 1. Create Google OAuth Credentials

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select existing one
3. Navigate to "APIs & Services" → "Credentials"
4. Click "Create Credentials" → "OAuth 2.0 Client ID"
5. Configure OAuth consent screen:
   - User Type: External
   - App name: GDE Edu AI
   - User support email: your email
   - Scopes: email, profile, openid
6. Create OAuth Client ID:
   - Application type: Web application
   - Name: GDE Edu AI Web Client
   - Authorized redirect URIs:
     - `https://localhost:7294/signin-google` (development)
     - `https://yourdomain.com/signin-google` (production)

### 2. Configure Application

Update `appsettings.json`:
```json
{
  "Authentication": {
    "Google": {
      "ClientId": "YOUR_CLIENT_ID.apps.googleusercontent.com",
      "ClientSecret": "YOUR_CLIENT_SECRET"
    }
  }
}
```

For production, use environment variables or Azure Key Vault:
```bash
export Authentication__Google__ClientId="your-client-id"
export Authentication__Google__ClientSecret="your-secret"
```

### 3. Install Required Package

Add to your `.csproj` file:
```xml
<PackageReference Include="Microsoft.AspNetCore.Authentication.Google" Version="8.0.0" />
```

Or via CLI:
```bash
dotnet add package Microsoft.AspNetCore.Authentication.Google
```

## Security Considerations

1. **Client Secret Protection**
   - Never commit secrets to source control
   - Use environment variables or secret managers
   - Rotate secrets periodically

2. **HTTPS Only**
   - OAuth requires HTTPS in production
   - Google rejects non-HTTPS redirect URIs

3. **CSRF Protection**
   - Built-in state parameter validation
   - Cookie-based anti-forgery tokens

4. **Token Storage**
   - Access tokens stored in secure cookies
   - Refresh tokens handled server-side only

5. **Scope Minimization**
   - Only request necessary scopes (email, profile)
   - Additional scopes require re-consent

## Testing OAuth Flow

### Development Testing
1. Start the application: `dotnet run`
2. Navigate to `/signin`
3. Click "Bejelentkezés Google fiókkal"
4. Authenticate with your Google account
5. Verify redirection to `/dashboard`
6. Check that profile information is populated

### Debugging Tips
1. Check browser console for redirect errors
2. Verify redirect URI matches Google Console exactly
3. Ensure ClientId and ClientSecret are correct
4. Test with multiple Google accounts
5. Clear cookies between tests

## Common Issues

### 1. "Redirect URI Mismatch"
**Solution:** Ensure redirect URI in Google Console matches exactly:
- Protocol (https)
- Domain
- Port (for localhost)
- Path (/signin-google)

### 2. "Access Denied" Error
**Solution:**
- Verify OAuth consent screen is published
- Check if test users are added (for unpublished apps)
- Ensure required scopes are requested

### 3. Claims Not Available
**Solution:**
- Verify scopes include "profile" and "email"
- Check `SaveTokens = true` in configuration
- Ensure ID token is being received

## Backend API Requirements

Your backend API should support:

1. **Google OAuth User Lookup**
```csharp
// Check if user exists by email
// If password starts with "GOOGLE_OAUTH_", validate Google ID
```

2. **Automatic User Creation**
```csharp
// Create user with Google OAuth marker
// Set Active = true (skip email verification)
// Populate profile from Google claims
```

3. **JWT Token Generation**
```csharp
// Generate JWT with user claims
// Include roles, email, name
// Return LoginResultModel with token
```

## Additional Resources

- [Google OAuth 2.0 Documentation](https://developers.google.com/identity/protocols/oauth2)
- [ASP.NET Core Authentication](https://docs.microsoft.com/aspnet/core/security/authentication/)
- [OpenID Connect Claims](https://openid.net/specs/openid-connect-core-1_0.html#Claims)
