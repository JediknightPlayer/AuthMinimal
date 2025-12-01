# Google OAuth Usage Examples

## Extracting and Using Google Profile Data

This document provides practical examples of how to extract and use Google OAuth data in your Blazor application.

## Example 1: Display Google Profile Picture

### Option A: Store in Database (Recommended)

When user signs in with Google, store the profile picture URL in the database:

```csharp
// In GoogleCallback.razor
private async Task HandleGoogleCallback()
{
    // Extract claims
    var picture = principal?.FindFirst("picture")?.Value;

    // Save to user profile in database
    var userData = new UserDataModel
    {
        ProfilePictureUrl = picture,
        // ... other fields
    };
}
```

Then display it in components:

```razor
@if (!string.IsNullOrEmpty(loggedUser.UserData.ProfilePictureUrl))
{
    <MudAvatar Size="Size.Medium">
        <MudImage Src="@loggedUser.UserData.ProfilePictureUrl"
                  Alt="@loggedUser.FirstName" />
    </MudAvatar>
}
else
{
    <MudAvatar Size="Size.Medium">
        <MudImage Src="img/profile.png" Alt="Default Profile" />
    </MudAvatar>
}
```

### Option B: Extract from Claims (Real-time)

Extract profile picture from authentication state:

```razor
@page "/profile"
@attribute [Authorize]
@inject AuthenticationStateProvider authStateProvider

<MudCard>
    <MudCardHeader>
        <MudAvatar Size="Size.Large">
            <MudImage Src="@profilePicture" Alt="Profile" />
        </MudAvatar>
        <MudCardHeaderContent>
            <MudText Typo="Typo.h6">@userName</MudText>
            <MudText Typo="Typo.body2">@userEmail</MudText>
        </MudCardHeaderContent>
    </MudCardHeader>
</MudCard>

@code {
    private string profilePicture = "img/profile.png";
    private string userName;
    private string userEmail;

    protected override async Task OnInitializedAsync()
    {
        var authState = await authStateProvider.GetAuthenticationStateAsync();
        var user = authState.User;

        if (user.Identity.IsAuthenticated)
        {
            userName = user.FindFirst(ClaimTypes.Name)?.Value ?? "User";
            userEmail = user.FindFirst(ClaimTypes.Email)?.Value ?? "";

            // Extract profile picture from custom claim
            var pictureClaim = user.FindFirst("picture")?.Value;
            if (!string.IsNullOrEmpty(pictureClaim))
            {
                profilePicture = pictureClaim;
            }
        }
    }
}
```

## Example 2: Pre-fill Registration Form

Auto-populate registration form with Google data:

```razor
@page "/complete-profile"
@attribute [Authorize]

<MudForm @ref="form" Model="@model">
    <MudText Typo="Typo.h5">Complete Your Profile</MudText>

    <MudAvatar Size="Size.Large" Class="my-4">
        <MudImage Src="@model.ProfilePicture" />
    </MudAvatar>

    <MudTextField @bind-Value="model.FirstName"
                  Label="First Name"
                  ReadOnly="@fromGoogle" />

    <MudTextField @bind-Value="model.LastName"
                  Label="Last Name"
                  ReadOnly="@fromGoogle" />

    <MudTextField @bind-Value="model.Email"
                  Label="Email"
                  ReadOnly="true" />

    <MudDatePicker @bind-Date="model.BirthDate"
                   Label="Birth Date" />

    <MudButton OnClick="SaveProfile"
               Color="Color.Primary">
        Save Profile
    </MudButton>
</MudForm>

@code {
    private MudForm form;
    private ProfileModel model = new();
    private bool fromGoogle = false;

    protected override async Task OnInitializedAsync()
    {
        var authState = await authStateProvider.GetAuthenticationStateAsync();
        var user = authState.User;

        if (user.Identity.IsAuthenticated)
        {
            // Check if user logged in with Google
            var googleId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            fromGoogle = !string.IsNullOrEmpty(googleId);

            // Pre-fill from Google claims
            model.FirstName = user.FindFirst(ClaimTypes.GivenName)?.Value ?? "";
            model.LastName = user.FindFirst(ClaimTypes.Surname)?.Value ?? "";
            model.Email = user.FindFirst(ClaimTypes.Email)?.Value ?? "";
            model.ProfilePicture = user.FindFirst("picture")?.Value ?? "img/profile.png";
            model.EmailVerified = user.FindFirst("email_verified")?.Value == "true";
        }
    }

    private async Task SaveProfile()
    {
        await form.Validate();
        if (form.IsValid)
        {
            // Save to database
            await userService.UpdateProfile(model);
        }
    }

    public class ProfileModel
    {
        public string FirstName { get; set; }
        public string LastName { get; set; }
        public string Email { get; set; }
        public DateTime? BirthDate { get; set; }
        public string ProfilePicture { get; set; }
        public bool EmailVerified { get; set; }
    }
}
```

## Example 3: Conditional UI Based on OAuth Provider

Show different UI elements based on login method:

```razor
<AuthorizeView>
    <Authorized>
        @if (IsGoogleUser())
        {
            <MudAlert Severity="Severity.Info">
                Signed in with Google
                <MudButton Size="Size.Small"
                           OnClick="ManageGoogleAccount">
                    Manage Google Account
                </MudButton>
            </MudAlert>
        }
        else
        {
            <MudAlert Severity="Severity.Info">
                Signed in with email
                <MudButton Size="Size.Small"
                           OnClick="ChangePassword">
                    Change Password
                </MudButton>
            </MudAlert>
        }
    </Authorized>
</AuthorizeView>

@code {
    private bool IsGoogleUser()
    {
        var authState = await authStateProvider.GetAuthenticationStateAsync();
        var user = authState.User;

        // Check for Google-specific claim
        var googleId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        return !string.IsNullOrEmpty(googleId) &&
               user.FindFirst("picture") != null;
    }

    private void ManageGoogleAccount()
    {
        navigationManager.NavigateTo("https://myaccount.google.com", true);
    }
}
```

## Example 4: Sync Profile Picture Periodically

Refresh Google profile picture on each login:

```csharp
// In GoogleCallback.razor - HandleGoogleCallback method

private async Task HandleGoogleCallback()
{
    // ... extract claims ...

    var picture = principal?.FindFirst("picture")?.Value;

    // Check if user already exists
    var existingUser = await userService.GetUserByEmail(email);

    if (existingUser != null)
    {
        // Update profile picture if it changed
        if (existingUser.UserData.ProfilePictureUrl != picture)
        {
            existingUser.UserData.ProfilePictureUrl = picture;
            existingUser.UserData.LastProfileSync = DateTime.UtcNow;
            await userService.UpdateUser(existingUser);
        }
    }
    else
    {
        // Create new user with profile picture
        var newUser = new UserModel
        {
            // ... other fields ...
            UserData = new UserDataModel
            {
                ProfilePictureUrl = picture,
                LastProfileSync = DateTime.UtcNow
            }
        };
        await userService.AddUser(newUser);
    }
}
```

## Example 5: Custom Claims in Your App

Add custom claims based on Google data:

```csharp
// In CustomAuthentication.cs - GetClaimsIdentity method

private ClaimsIdentity GetClaimsIdentity(LoginUserModel user)
{
    var claims = new List<Claim>
    {
        new Claim(ClaimTypes.Sid, user.Id.ToString()),
        new Claim(ClaimTypes.Name, user.FirstName),
        new Claim(ClaimTypes.Email, user.Email),

        // Add custom claims
        new Claim("profile_picture", user.UserData.ProfilePictureUrl ?? ""),
        new Claim("email_verified", user.UserData.EmailVerified.ToString()),
        new Claim("oauth_provider", user.Guid.StartsWith("GOOGLE_") ? "Google" : "Local"),
        new Claim("last_sync", user.UserData.LastProfileSync?.ToString("o") ?? "")
    };

    foreach (var role in user.Roles)
    {
        claims.Add(new Claim(ClaimTypes.Role, role.Name));
    }

    return new ClaimsIdentity(claims, "CustomAuth");
}
```

Then access in components:

```razor
@code {
    private string GetProfilePicture()
    {
        var authState = await authStateProvider.GetAuthenticationStateAsync();
        return authState.User.FindFirst("profile_picture")?.Value ?? "img/profile.png";
    }

    private bool IsEmailVerified()
    {
        var authState = await authStateProvider.GetAuthenticationStateAsync();
        return authState.User.FindFirst("email_verified")?.Value == "true";
    }

    private string GetOAuthProvider()
    {
        var authState = await authStateProvider.GetAuthenticationStateAsync();
        return authState.User.FindFirst("oauth_provider")?.Value ?? "Local";
    }
}
```

## Example 6: Handle Profile Picture Errors

Gracefully handle missing or broken profile pictures:

```razor
<MudAvatar Size="Size.Medium">
    <MudImage Src="@GetSafeProfilePicture()"
              Alt="@userName"
              @onerror="HandleImageError" />
</MudAvatar>

@code {
    private string profilePicture;
    private string fallbackPicture = "img/profile.png";
    private bool imageError = false;

    private string GetSafeProfilePicture()
    {
        if (imageError || string.IsNullOrEmpty(profilePicture))
        {
            return fallbackPicture;
        }
        return profilePicture;
    }

    private void HandleImageError()
    {
        imageError = true;
        StateHasChanged();
    }

    protected override async Task OnInitializedAsync()
    {
        var authState = await authStateProvider.GetAuthenticationStateAsync();
        var user = authState.User;

        profilePicture = user.FindFirst("picture")?.Value;

        // Validate URL
        if (!string.IsNullOrEmpty(profilePicture) &&
            !Uri.IsWellFormedUriString(profilePicture, UriKind.Absolute))
        {
            profilePicture = fallbackPicture;
        }
    }
}
```

## Example 7: Download and Store Profile Picture Locally

Instead of using Google's URL, download and store the image:

```csharp
// In GoogleCallback.razor

private async Task<string> DownloadAndStoreProfilePicture(string pictureUrl, string userId)
{
    if (string.IsNullOrEmpty(pictureUrl))
        return null;

    try
    {
        using var httpClient = new HttpClient();
        var imageBytes = await httpClient.GetByteArrayAsync(pictureUrl);

        // Save to your storage (Azure Blob, local file system, etc.)
        var fileName = $"profile_{userId}_{DateTime.UtcNow.Ticks}.jpg";
        var localPath = $"wwwroot/uploads/profiles/{fileName}";

        await File.WriteAllBytesAsync(localPath, imageBytes);

        return $"/uploads/profiles/{fileName}";
    }
    catch (Exception ex)
    {
        // Log error and return null
        return null;
    }
}

// Usage
var localPicturePath = await DownloadAndStoreProfilePicture(picture, email);
newUser.UserData.ProfilePictureUrl = localPicturePath ?? picture;
```

## Example 8: Show Google Account Badge

Display a badge for Google-authenticated users:

```razor
<MudCard>
    <MudCardHeader>
        <MudAvatar>
            <MudImage Src="@profilePicture" />
        </MudAvatar>
        <MudCardHeaderContent>
            <div class="d-flex align-center">
                <MudText Typo="Typo.h6">@userName</MudText>
                @if (isGoogleUser)
                {
                    <MudChip Size="Size.Small"
                             Color="Color.Info"
                             Icon="@Icons.Custom.Brands.Google"
                             Class="ml-2">
                        Google
                    </MudChip>
                }
            </div>
            <MudText Typo="Typo.body2">@userEmail</MudText>
        </MudCardHeaderContent>
    </MudCardHeader>
</MudCard>

@code {
    private bool isGoogleUser;
    private string profilePicture;
    private string userName;
    private string userEmail;

    protected override async Task OnInitializedAsync()
    {
        var authState = await authStateProvider.GetAuthenticationStateAsync();
        var user = authState.User;

        isGoogleUser = user.FindFirst("picture") != null;
        profilePicture = user.FindFirst("picture")?.Value ?? "img/profile.png";
        userName = user.FindFirst(ClaimTypes.Name)?.Value;
        userEmail = user.FindFirst(ClaimTypes.Email)?.Value;
    }
}
```

## Best Practices

1. **Always provide fallback images**
   - Google profile URLs can expire
   - Users may revoke access
   - Network errors can occur

2. **Cache profile pictures**
   - Download and store locally for better performance
   - Reduces dependency on Google's CDN

3. **Respect user privacy**
   - Allow users to hide their Google profile picture
   - Provide option to upload custom picture

4. **Handle claims safely**
   - Always check for null values
   - Validate URLs before using them
   - Use try-catch for claim extraction

5. **Keep profile data in sync**
   - Periodically refresh profile information
   - Update on each login
   - Allow manual refresh option

6. **Security considerations**
   - Validate profile picture URLs
   - Sanitize all user input
   - Use HTTPS for all images
   - Implement CSP headers
