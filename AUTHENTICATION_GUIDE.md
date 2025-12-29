# Authentication Guide for SleepQ gRPC-Web Service

## Overview

This guide explains how to implement secure authentication for your SwiftUI app communicating with the gRPC-Web backend.

## ❌ What NOT to Do

**NEVER pass user IDs in request bodies:**

```protobuf
// ❌ BAD - Don't do this!
message GetTodayRequest {
  string user_id = 1;  // Security risk!
}
```

**Why?** Clients can easily modify this to impersonate other users.

---

## ✅ Recommended Approach: Token-Based Authentication

### Architecture

```
┌─────────────┐         ┌──────────────┐         ┌──────────────┐
│  SwiftUI    │  Login  │   Backend    │ Validate│  Database    │
│   Client    ├────────>│   Service    ├────────>│              │
│             │<────────┤              │         │              │
└──────┬──────┘  Token  └──────────────┘         └──────────────┘
       │
       │ Subsequent Requests
       │ Header: Authorization: Bearer <token>
       │
       └────────> GetToday()
                 GetIssueDetail()
                 etc.
```

---

## Implementation Options

### **Option 1: JWT Tokens (Production - Recommended)**

#### **Backend Setup**

1. **Add NuGet Packages**:
```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package System.IdentityModel.Tokens.Jwt
```

2. **Configure JWT in Program.cs**:
```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

// Add JWT Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!))
        };
    });

builder.Services.AddAuthorization();

// ... rest of configuration

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
```

3. **Add to appsettings.json**:
```json
{
  "Jwt": {
    "Key": "your-secret-key-min-32-characters-long!",
    "Issuer": "SleepQBackend",
    "Audience": "SleepQClients",
    "ExpirationMinutes": 1440
  }
}
```

#### **SwiftUI Client Setup**

1. **Install gRPC-Swift**:
```swift
dependencies: [
    .package(url: "https://github.com/grpc/grpc-swift.git", from: "1.21.0")
]
```

2. **Create Authentication Manager**:
```swift
import Foundation
import GRPC
import NIO

class AuthenticationManager: ObservableObject {
    @Published var authToken: String?
    @Published var isAuthenticated = false

    private let serverURL = "https://your-backend.azurewebsites.net"

    func login(email: String, password: String) async throws {
        // Call your login endpoint (you'll need to create this)
        let loginRequest = LoginRequest.with {
            $0.email = email
            $0.password = password
        }

        // Make gRPC call to login service
        // Store the returned JWT token
        let response = try await performLogin(loginRequest)

        DispatchQueue.main.async {
            self.authToken = response.token
            self.isAuthenticated = true
            // Store token in Keychain for persistence
            KeychainHelper.save(token: response.token)
        }
    }

    func logout() {
        authToken = nil
        isAuthenticated = false
        KeychainHelper.deleteToken()
    }
}
```

3. **Create Authenticated gRPC Client**:
```swift
import GRPC
import NIO

class SleepQClient {
    private let group: EventLoopGroup
    private let channel: GRPCChannel
    private let authManager: AuthenticationManager

    init(authManager: AuthenticationManager) {
        self.authManager = authManager
        self.group = PlatformSupport.makeEventLoopGroup(loopCount: 1)

        // Configure channel for gRPC-Web
        self.channel = try! GRPCChannelPool.with(
            target: .host("your-backend.azurewebsites.net", port: 443),
            transportSecurity: .tls(GRPCTLSConfiguration.makeClientDefault()),
            eventLoopGroup: group
        )
    }

    // Create client with authentication interceptor
    func createSleepServiceClient() -> Zaicontracts_Sleep_SleepServiceAsyncClient {
        return Zaicontracts_Sleep_SleepServiceAsyncClient(
            channel: channel,
            defaultCallOptions: authenticatedCallOptions()
        )
    }

    private func authenticatedCallOptions() -> CallOptions {
        var callOptions = CallOptions()

        // Add JWT token to metadata
        if let token = authManager.authToken {
            callOptions.customMetadata.add(
                name: "authorization",
                value: "Bearer \(token)"
            )
        }

        return callOptions
    }
}
```

4. **Use in SwiftUI View**:
```swift
import SwiftUI

struct TodayView: View {
    @EnvironmentObject var authManager: AuthenticationManager
    @State private var todayData: GetTodayResponse?

    private let grpcClient: SleepQClient

    init(grpcClient: SleepQClient) {
        self.grpcClient = grpcClient
    }

    var body: some View {
        VStack {
            if let data = todayData {
                Text("Sleep Quality: \(data.sleepQuality)")
                Text("Confidence: \(String(format: "%.1f%%", data.confidence * 100))")

                ForEach(data.topIssues, id: \.element) { issue in
                    IssueCard(issue: issue)
                }
            } else {
                ProgressView("Loading...")
            }
        }
        .task {
            await loadTodayData()
        }
    }

    private func loadTodayData() async {
        do {
            let client = grpcClient.createSleepServiceClient()
            let request = GetTodayRequest()  // No user_id needed!

            todayData = try await client.getToday(request)
        } catch {
            print("Error loading today data: \(error)")
        }
    }
}
```

---

### **Option 2: Custom Header (Development/Testing Only)**

For development and testing, you can use a simple custom header:

#### **SwiftUI Client**:
```swift
private func testCallOptions(userId: String) -> CallOptions {
    var callOptions = CallOptions()
    callOptions.customMetadata.add(
        name: "x-user-id",
        value: userId
    )
    return callOptions
}

// Usage
let client = Zaicontracts_Sleep_SleepServiceAsyncClient(
    channel: channel,
    defaultCallOptions: testCallOptions(userId: "your-test-user-id")
)
```

⚠️ **Warning**: Only use this for development. Never use in production!

---

### **Option 3: OAuth 2.0 / OpenID Connect (Enterprise)**

For enterprise applications, integrate with identity providers:

- **Azure AD / Microsoft Entra ID**
- **Auth0**
- **Firebase Authentication**
- **Okta**

These provide:
- Single Sign-On (SSO)
- Multi-factor authentication (MFA)
- Enterprise user management
- Token refresh handling

---

## Security Best Practices

### ✅ Do:
1. **Use HTTPS/TLS** for all communication
2. **Validate tokens server-side** on every request
3. **Store tokens securely** in iOS Keychain
4. **Implement token refresh** for long-lived sessions
5. **Use short token expiration** (15-60 minutes for access tokens)
6. **Log authentication events** for audit trails
7. **Implement rate limiting** on authentication endpoints

### ❌ Don't:
1. **Don't trust client-provided user IDs**
2. **Don't store tokens in UserDefaults** (use Keychain)
3. **Don't send tokens in URL parameters**
4. **Don't skip token validation server-side**
5. **Don't use weak signing keys** (min 256-bit for HMAC)
6. **Don't log tokens** in plain text

---

## Testing Your Implementation

### Test 1: Valid Token
```bash
# Generate a test token (you'll need to create an endpoint for this)
TOKEN="eyJhbGc..."

# Make gRPC-Web call with token
grpcurl -H "Authorization: Bearer $TOKEN" \
  your-backend.azurewebsites.net:443 \
  zaicontracts.sleep.SleepService/GetToday
```

### Test 2: Invalid Token
```bash
# Should fail with Unauthenticated error
grpcurl -H "Authorization: Bearer invalid-token" \
  your-backend.azurewebsites.net:443 \
  zaicontracts.sleep.SleepService/GetToday
```

### Test 3: No Token
```bash
# Should fail with Unauthenticated error
grpcurl your-backend.azurewebsites.net:443 \
  zaicontracts.sleep.SleepService/GetToday
```

---

## Migration Path

### Phase 1: Development (Current)
- Use `x-user-id` header for testing
- Hardcode test user IDs
- Focus on feature development

### Phase 2: Alpha/Beta
- Implement JWT authentication
- Add login/registration endpoints
- Test with real users

### Phase 3: Production
- Enable token refresh
- Add OAuth providers (optional)
- Implement proper session management
- Add MFA support

---

## Common Issues & Solutions

### Issue: "User is not authenticated" error

**Solution**: Check that:
1. Token is being sent in `Authorization` header
2. Token format is correct: `Bearer <token>`
3. Token hasn't expired
4. Token signature is valid

### Issue: Token works in Postman but not SwiftUI

**Solution**: Verify metadata is being added correctly:
```swift
// Debug: Print metadata being sent
print("Sending metadata: \(callOptions.customMetadata)")
```

### Issue: gRPC-Web CORS errors

**Solution**: Add CORS middleware in Program.cs:
```csharp
app.UseCors(policy => policy
    .AllowAnyOrigin()
    .AllowAnyMethod()
    .AllowAnyHeader()
    .WithExposedHeaders("Grpc-Status", "Grpc-Message", "Grpc-Encoding"));
```

---

## Next Steps

1. **Implement login endpoint** (create LoginService.proto)
2. **Add user registration**
3. **Implement token refresh**
4. **Add password reset functionality**
5. **Integrate with Apple Sign In** (for iOS)
6. **Add biometric authentication** (Face ID/Touch ID)

For questions or issues, refer to the main documentation or contact the development team.
