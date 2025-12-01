# Google OAuth 2.0 Quick Reference

## Setup Checklist (5 minutes)

### 1. Install Package
```bash
dotnet add package Microsoft.AspNetCore.Authentication.Google
```

### 2. Configure appsettings.json
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

### 3. Get Google Credentials
1. Go to: https://console.cloud.google.com/
2. Create project → APIs & Services → Credentials
3. Create OAuth 2.0 Client ID
4. Add redirect URI: `https://localhost:7294/signin-google`

### 4. Test
- Run app → Navigate to `/signin`
- Click "Bejelentkezés Google fiókkal"
- Sign in with Google
- Should redirect to `/dashboard`

## Key Files

| File | Purpose |
|------|---------|
| `Program.cs` | OAuth configuration |
| `Signin.razor` | Login button |
| `GoogleCallback.razor` | OAuth callback handler |
| `CustomAuthentication.cs` | Claims integration |

## Common Code Snippets

### Extract Claims in Component
```csharp
protected override async Task OnInitializedAsync()
{
    var authState = await authStateProvider.GetAuthenticationStateAsync();
    var user = authState.User;

    var email = user.FindFirst(ClaimTypes.Email)?.Value;
    var name = user.FindFirst(ClaimTypes.Name)?.Value;
    var picture = user.FindFirst("picture")?.Value;
}
```

### Display Profile Picture
```razor
<MudAvatar Size="Size.Medium">
    <MudImage Src="@(user.FindFirst("picture")?.Value ?? "img/profile.png")" />
</MudAvatar>
```

### Check if Google User
```csharp
private bool IsGoogleUser()
{
    return user.FindFirst("picture") != null;
}
```

## Available Claims

```csharp
ClaimTypes.Email           // user@gmail.com
ClaimTypes.Name            // John Doe
ClaimTypes.GivenName       // John
ClaimTypes.Surname         // Doe
ClaimTypes.NameIdentifier  // Google User ID
"picture"                  // Profile picture URL
"email_verified"           // true/false
```

## OAuth Flow (Simplified)

```
Click Button → Redirect to Google → User Approves
→ Redirect to /signin-google → Extract Claims
→ Login/Create User → Set Auth State → Dashboard
```

## Troubleshooting (30 seconds)

| Problem | Solution |
|---------|----------|
| Redirect URI mismatch | Check Google Console matches exactly |
| 401 Unauthorized | Verify ClientId/ClientSecret |
| Claims are null | Add scopes: `profile`, `email` |
| No profile picture | Check `"picture"` claim |
| Can't create user | Backend must handle `GOOGLE_OAUTH_` prefix |

## Test Commands

```bash
# Check if package is installed
dotnet list package | grep Google

# Run application
dotnet run

# View OAuth configuration
cat appsettings.json | grep -A5 "Authentication"
```

## Environment Variables (Production)

```bash
export Authentication__Google__ClientId="your-client-id"
export Authentication__Google__ClientSecret="your-secret"
```

## Security Checklist

- [ ] Use HTTPS in production
- [ ] Store secrets in Key Vault (not appsettings.json)
- [ ] Verify redirect URIs in Google Console
- [ ] Enable SaveTokens only if needed
- [ ] Implement token refresh for long sessions
- [ ] Add logging for OAuth failures

## Quick Debug

```csharp
// In GoogleCallback.razor - add to HandleGoogleCallback()
Console.WriteLine($"Email: {email}");
Console.WriteLine($"Name: {name}");
Console.WriteLine($"Picture: {picture}");
Console.WriteLine($"Claims count: {principal.Claims.Count()}");
```

## Production Configuration

```json
{
  "Authentication": {
    "Google": {
      "ClientId": "prod-client-id.apps.googleusercontent.com",
      "ClientSecret": "prod-secret"
    }
  }
}
```

Add to Google Console:
- `https://yourdomain.com/signin-google`

## Common Customizations

### Change Button Text
```razor
<MudButton OnClick="@SignInWithGoogle">
    Your Custom Text
</MudButton>
```

### Custom Redirect After Login
```csharp
// In GoogleCallback.razor
navigationManager.NavigateTo("/custom-page");
```

### Store Additional Data
```csharp
var newUser = new UserModel
{
    Email = email,
    FirstName = givenName,
    LastName = surname,
    UserData = new UserDataModel
    {
        ProfilePictureUrl = picture,
        EmailVerified = emailVerified,
        OAuthProvider = "Google",
        GoogleUserId = nameIdentifier
    }
};
```

## Need More Info?

- **Full Setup**: See `GOOGLE_OAUTH_SETUP.md`
- **Code Examples**: See `OAUTH_USAGE_EXAMPLES.md`
- **Overview**: See `OAUTH_IMPLEMENTATION_SUMMARY.md`
