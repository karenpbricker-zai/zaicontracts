# SwiftUI + gRPC-Web Quick Start Guide

## 🎯 Answer: No Login Page Needed!

**You can create users automatically without requiring login.**

---

## ✅ Recommended Solution: Anonymous Auto-Login

Your sleep app will:
1. ✅ Create user account automatically on first launch
2. ✅ Start tracking immediately (no login screen!)
3. ✅ Optionally allow "Sign in with Apple" later for multi-device sync

---

## 📱 SwiftUI Implementation (5 Steps)

### Step 1: Create AuthManager

```swift
// AuthManager.swift
import Foundation
import SwiftUI

class AuthManager: ObservableObject {
    @Published var userId: String?
    @Published var isReady = false

    private let keychainKey = "sleepq_user_id"

    init() {
        Task {
            await initializeUser()
        }
    }

    @MainActor
    func initializeUser() async {
        // Check if user already exists
        if let existingUserId = KeychainHelper.load(key: keychainKey) {
            self.userId = existingUserId
            self.isReady = true
            print("✅ Found existing user: \(existingUserId)")
            return
        }

        // Create new anonymous user
        await createAnonymousUser()
    }

    private func createAnonymousUser() async {
        do {
            let deviceId = UIDevice.current.identifierForVendor?.uuidString ?? UUID().uuidString

            let request = CreateAnonymousUserRequest.with {
                $0.deviceID = deviceId
            }

            let client = GrpcClientFactory.shared.createAuthClient()
            let response = try await client.createAnonymousUser(request)

            await MainActor.run {
                self.userId = response.userID
                self.isReady = true
                KeychainHelper.save(key: keychainKey, value: response.userID)
                print("✅ Created new user: \(response.userID)")
            }
        } catch {
            print("❌ Error creating user: \(error)")
        }
    }
}
```

### Step 2: Create Keychain Helper

```swift
// KeychainHelper.swift
import Foundation
import Security

struct KeychainHelper {
    static func save(key: String, value: String) {
        let data = value.data(using: .utf8)!

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data
        ]

        SecItemDelete(query as CFDictionary)
        SecItemAdd(query as CFDictionary, nil)
    }

    static func load(key: String) -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true
        ]

        var result: AnyObject?
        SecItemCopyMatching(query as CFDictionary, &result)

        guard let data = result as? Data,
              let value = String(data: data, encoding: .utf8) else {
            return nil
        }

        return value
    }
}
```

### Step 3: Create gRPC Client Factory

```swift
// GrpcClientFactory.swift
import GRPC
import NIO

class GrpcClientFactory {
    static let shared = GrpcClientFactory()

    private let group: EventLoopGroup
    private let channel: GRPCChannel

    private init() {
        self.group = PlatformSupport.makeEventLoopGroup(loopCount: 1)

        // For local testing
        self.channel = try! GRPCChannelPool.with(
            target: .host("localhost", port: 8080),
            transportSecurity: .plaintext,
            eventLoopGroup: group
        )

        // For production (Azure)
        // self.channel = try! GRPCChannelPool.with(
        //     target: .host("your-backend.azurewebsites.net", port: 443),
        //     transportSecurity: .tls(GRPCTLSConfiguration.makeClientDefault()),
        //     eventLoopGroup: group
        // )
    }

    func createSleepServiceClient(userId: String) -> Zaicontracts_Sleep_SleepServiceAsyncClient {
        var callOptions = CallOptions()

        // Add user ID to metadata
        callOptions.customMetadata.add(name: "x-user-id", value: userId)

        return Zaicontracts_Sleep_SleepServiceAsyncClient(
            channel: channel,
            defaultCallOptions: callOptions
        )
    }
}
```

### Step 4: Update Your App Entry Point

```swift
// SleepQApp.swift
import SwiftUI

@main
struct SleepQApp: App {
    @StateObject private var authManager = AuthManager()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(authManager)
        }
    }
}
```

### Step 5: Use in Your Views

```swift
// ContentView.swift
struct ContentView: View {
    @EnvironmentObject var authManager: AuthManager

    var body: some View {
        Group {
            if authManager.isReady {
                MainTabView()
            } else {
                ProgressView("Setting up...")
            }
        }
    }
}

// TodayView.swift
struct TodayView: View {
    @EnvironmentObject var authManager: AuthManager
    @State private var todayData: GetTodayResponse?

    var body: some View {
        VStack {
            if let data = todayData {
                Text("Sleep Quality: \(data.sleepQuality)")
                // ... rest of your UI
            }
        }
        .task {
            await loadData()
        }
    }

    private func loadData() async {
        guard let userId = authManager.userId else { return }

        let client = GrpcClientFactory.shared.createSleepServiceClient(userId: userId)

        do {
            todayData = try await client.getToday(GetTodayRequest())
        } catch {
            print("Error: \(error)")
        }
    }
}
```

---

## 🔧 Backend Setup (Already Done!)

Your backend is already configured to:
✅ Accept `x-user-id` header
✅ Extract user ID from context
✅ Prevent user impersonation

Just need to add the `CreateAnonymousUser` endpoint:

```csharp
// Add to your proto files (create auth_service.proto)
syntax = "proto3";
package zaicontracts.auth;

message CreateAnonymousUserRequest {
  string device_id = 1;
}

message CreateAnonymousUserResponse {
  string user_id = 1;
}

service AuthService {
  rpc CreateAnonymousUser(CreateAnonymousUserRequest) returns (CreateAnonymousUserResponse);
}
```

---

## 🚀 How It Works

```
User Opens App
    ↓
Check Keychain for user_id
    ↓
    ├─ Found? → Use existing user
    │            ↓
    │         Start app!
    │
    └─ Not found? → Call CreateAnonymousUser()
                    ↓
                 Backend creates user
                    ↓
                 Save user_id to Keychain
                    ↓
                 Start app!
```

**User sees**: Just the app starting up (0.5 seconds)
**No login screen!** 🎉

---

## 📊 What User Sees vs What Happens

| User Experience | Behind the Scenes |
|----------------|-------------------|
| Opens app for first time | Auto-creates account with device ID |
| Sees main screen immediately | Saves user ID in Keychain |
| Starts tracking sleep | All data tied to that user ID |
| Next time opens app | Loads user ID from Keychain |
| Works offline | Local Keychain storage |

---

## 🔐 Security Features

✅ **User ID stored in Keychain** (encrypted by iOS)
✅ **Device ID prevents duplicates** (same device always gets same user)
✅ **Server validates all requests** (via GrpcAuthInterceptor)
✅ **Cannot impersonate other users** (user ID is server-controlled)

---

## 🎁 Bonus: Add "Sign in with Apple" Later

Add this to SettingsView to let users sync across devices:

```swift
struct SettingsView: View {
    @EnvironmentObject var authManager: AuthManager

    var body: some View {
        List {
            Section {
                Button(action: {
                    // TODO: Implement Apple Sign In
                }) {
                    Label("Sign in to sync across devices", systemImage: "icloud")
                }
            } footer: {
                Text("Your data is currently on this device only")
            }
        }
    }
}
```

---

## 🐛 Troubleshooting

### "User is not authenticated" error

**Check**:
1. Is `x-user-id` header being sent?
2. Is it a valid GUID?
3. Print the metadata being sent:
   ```swift
   print("Metadata: \(callOptions.customMetadata)")
   ```

### User ID changes every launch

**Issue**: Not saving to Keychain properly

**Fix**: Verify KeychainHelper.save() is called after user creation

### Can't connect to backend

**For local testing**:
```swift
// Use .plaintext for localhost
target: .host("localhost", port: 8080),
transportSecurity: .plaintext
```

**For Azure**:
```swift
// Use .tls for HTTPS
target: .host("your-backend.azurewebsites.net", port: 443),
transportSecurity: .tls(GRPCTLSConfiguration.makeClientDefault())
```

---

## 📚 Full Documentation

- **SWIFTUI_AUTH_OPTIONS.md** - All authentication options
- **AUTHENTICATION_GUIDE.md** - Backend setup details
- **AUTHENTICATION_SUMMARY.md** - Quick reference

---

## ✅ Summary

**You DO NOT need a login page!**

✅ Users can start immediately
✅ Account created automatically
✅ Data persists across app launches
✅ Optional sign-in later for multi-device

Your app will have the smoothest onboarding possible! 🚀
