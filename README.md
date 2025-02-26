# Easy-SwiftUI-Firebase-Authentication

This README provides instructions for implementing authentication in your SwiftUI application using Firebase Authentication with Apple, Google, and Email/Password sign-in methods.

Using [SwiftfulAuthenticating](https://github.com/SwiftfulThinking/SwiftfulAuthenticating) and [SwiftfulAuthenticatingFirebase](https://github.com/SwiftfulThinking/SwiftfulAuthenticatingFirebase).

## Requirements

- Xcode (latest version recommended)
- iOS project with SwiftUI
- Firebase account with a configured project

## Installation

### Step 1: Add Required Packages

In Xcode, go to File > Add Packages... and add the following packages:

```
https://github.com/SwiftfulThinking/SwiftfulAuthenticating.git
https://github.com/SwiftfulThinking/SwiftfulAuthenticatingFirebase.git
https://github.com/firebase/firebase-ios-sdk.git
```

When adding the Firebase package, select the following products:
- FirebaseAuth
- FirebaseCore

### Step 2: Configure Firebase

1. Create a Firebase project at [firebase.google.com](https://firebase.google.com/)
2. Register your iOS app in the Firebase console
   - Enter your bundle ID (e.g., com.yourdomain.yourapp)
   - Enter a nickname for your app (optional)
   - Enable Google Analytics if needed

3. Download the `GoogleService-Info.plist` file and add it to your Xcode project
   - Make sure to add it to all targets
   - Ensure "Copy items if needed" is checked

4. URL Types Configuration for Google Sign-In:
   - In Xcode, select your project file → Info tab → URL Types
   - Click the + button to add a new URL Type
   - Set the URL Schemes to the REVERSED_CLIENT_ID from your GoogleService-Info.plist

5. Enable Authentication Methods in Firebase Console:
   - Go to Firebase Console → Authentication → Sign-in method
   - Enable the authentication methods you want to use (Google, Apple, Email/Password)

6. For Google Sign-In:
   - Note your CLIENT_ID from GoogleService-Info.plist (needed in your auth code)

7. For Apple Sign-In:
   - Configure Sign in with Apple in your Apple Developer account
   - Select your project file → Signing & Capabilities → "+" → "Sign in with Apple"

## Core Implementation

### Step 1: Create Service Files

Create a `Services.swift` file:

```swift
//  Services.swift

import SwiftUI
import SwiftfulAuthenticating
import SwiftfulAuthenticatingFirebase

@MainActor
class AppServices {
    static func createAuthViewModel() -> AuthViewModel {
        let authService = FirebaseAuthService()
        return AuthViewModel(service: authService)
    }
}

@MainActor
class AuthService {
    static let shared = AuthService()
    
    let authManager = AuthManager(service: FirebaseAuthService())
    
    private init() {}
    
    static func getShared() async -> AuthService {
        await MainActor.run { shared }
    }
}
```

### Step 2: Create Authentication View Model

Create `AuthViewModel.swift`:

```swift
//  AuthViewModels.swift

import Foundation
import SwiftfulAuthenticating
import SwiftfulAuthenticatingFirebase
import FirebaseAuth

@Observable
@MainActor
class AuthViewModel: ObservableObject {
    // Initialize AuthManager directly
    private let authManager: AuthManager    
    var errorMessage = ""
    var showError = false
    
    private var refreshTrigger = false
    
    init(service: any SwiftfulAuthenticating.AuthService) {
        self.authManager = AuthManager(service: service)
    }
    
    // MARK: - User Properties
    
    var currentUser: UserAuthInfo? {
        authManager.auth
    }
    
    var isAuthenticated: Bool {
        // Using the auth property directly
        authManager.auth != nil
    }
    
    // MARK: - Authentication Methods
    
    func signInWithApple() async {
        do {
            let (userInfo, isNewUser) = try await authManager.signInApple()
            handleSuccessfulSignIn(userInfo: userInfo, isNewUser: isNewUser)
        } catch {
            handleAuthError(error)
        }
    }
    
    func signInWithGoogle(clientId: String) async {
        do {
            let (userInfo, isNewUser) = try await authManager.signInGoogle(GIDClientID: clientId)
            handleSuccessfulSignIn(userInfo: userInfo, isNewUser: isNewUser)
        } catch {
            handleAuthError(error)
        }
    }
    
    
    func signOut() async {
        do {
            try authManager.signOut()
            refreshTrigger.toggle()
        } catch {
            handleAuthError(error)
        }
    }
    
    // MARK: - Helper Methods
    
    private func handleSuccessfulSignIn(userInfo: UserAuthInfo, isNewUser: Bool) {
        refreshTrigger.toggle()
    }

    private func handleAuthError(_ error: Error) {
        errorMessage = error.localizedDescription
        showError = true
        print("Authentication error: \(error.localizedDescription)")
    }
}
```

### Step 3: Configure App Entry Point

Modify your `App.swift` file:

```swift
import SwiftUI
import Firebase
import FirebaseAuth
import SwiftfulAuthenticating
import SwiftfulAuthenticatingFirebase

class AppDelegate: NSObject, UIApplicationDelegate {
  func application(_ application: UIApplication,
                   didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey : Any]? = nil) -> Bool {
    FirebaseApp.configure()
    return true
  }
}

@main
struct YourApp: App { // Change this for your project
    @UIApplicationDelegateAdaptor(AppDelegate.self) var delegate
    
    @StateObject private var authViewModel = AuthViewModel(
        service: FirebaseAuthService()
    )
    
    var body: some Scene {
        WindowGroup {
            ContentView(viewModel: authViewModel)
        }
    }
}
```

### Step 4: Modify Content View

Modify your `ContentView.swift` file:

```swift
// ContentView.swift

import SwiftUI
import SwiftfulAuthenticating

struct ContentView: View {
    @StateObject var viewModel: AuthViewModel

    var body: some View {
        Group {
            if viewModel.isAuthenticated {
                // User is signed in
                HomeView(viewModel: viewModel) // Create this view for your project 
            } else {
                // User is not signed in
                AuthView(viewModel: viewModel) // Create this view for your project
            }
        }
    }
}
```

## View Implementation Code Snippets

### Authentication View

For your Auth View, add these properties to your view struct:

```swift
@StateObject var viewModel: AuthViewModel
private let googleClientId = "YOUR_CLIENT_ID" // Replace with your CLIENT_ID from GoogleService-Info.plist
```

Add these button actions:

```swift
// Sign in with Apple button
Button("Sign in with Apple") {
    Task {
        await viewModel.signInWithApple()
    }
}

// Sign in with Google button
Button("Sign in with Google") {
    Task {
        await viewModel.signInWithGoogle(clientId: googleClientId)
    }
}
```

Add this error alert modifier to your main container:

```swift
.alert("Authentication Error", isPresented: $viewModel.showError) {
    Button("OK") {}
} message: {
    Text(viewModel.errorMessage)
}
```

### Settings View

For your Settings View, simply add this property to your view struct:

```swift
@StateObject var viewModel: AuthViewModel
```

And add this code for your sign out button:

```swift
Button("Sign Out") {
    Task {
        await viewModel.signOut()
    }
}
```

## Email Authentication Implementation (not supported by SwiftfulAuthenticating -> with FirebaseAuth)

If you want to implement email/password authentication:

### Step 1: Create Email Manager

Create `EmailManager.swift`:

```swift
//  EmailManager.swift

import Foundation
import FirebaseAuth

struct AuthDataResultModel {
    let uid: String
    let email: String?
    let photoUrl: String?

    init(user: User) {
        self.uid = user.uid
        self.email = user.email
        self.photoUrl = user.photoURL?.absoluteString
    }
}

final class EmailManager {
    static let shared = EmailManager()
    
    private init() {}
    
    func createUser(email: String, password: String) async throws -> User {
        let authResult = try await Auth.auth().createUser(withEmail: email, password: password)
        return authResult.user
    }
    
    func signIn(email: String, password: String) async throws -> User {
        let authResult = try await Auth.auth().signIn(withEmail: email, password: password)
        return authResult.user
    }

    func resetPassword(email: String) async throws {
        try await Auth.auth().sendPasswordReset(withEmail: email)
    }
}
```

### Step 2: Create Email Authentication View Model

Create `AuthEmailViewModel.swift`:

```swift
// AuthEmailViewModel.swift

import Foundation
import FirebaseAuth

@MainActor
final class AuthEmailViewModel: ObservableObject {
    @Published var email = ""
    @Published var password = ""
    @Published var errorMessage = ""
    @Published var showError = false
    
    private let auth = Auth.auth()

    func signUp() async throws {
        guard !email.isEmpty, !password.isEmpty else {
            errorMessage = "Please fill in all fields"
            showError = true
            throw NSError(domain: "EmptyFields", code: 400)
        }
        
        try await auth.createUser(withEmail: email, password: password)
    }

    func signIn() async throws {
        guard !email.isEmpty, !password.isEmpty else {
            errorMessage = "Please fill in all fields"
            showError = true
            throw NSError(domain: "EmptyFields", code: 400)
        }
        
        try await auth.signIn(withEmail: email, password: password)
    }

    func resetPassword() async throws {
        guard let email = auth.currentUser?.email else {
            errorMessage = "No email found"
            showError = true
            throw NSError(domain: "NoEmailFound", code: 400)
        }

        try await EmailManager.shared.resetPassword(email: email)
    }
}
```

### Step 3: Email Authentication View Implementation

For your Email Authentication View, add this property to your view struct:

```swift
@ObservedObject var viewModel: AuthEmailViewModel
```

Link your text fields:

```swift
// Email TextField
TextField("Email", text: $viewModel.email)

// Password SecureField
SecureField("Password", text: $viewModel.password)
```

Add this button action for sign in/sign up:

```swift
Button("Sign In/Sign Up") {
    Task {
        do {
            try await viewModel.signUp()
            return
        } catch {
            print("Error signing up: \(error)")
        }

        do {
            try await viewModel.signIn()
            return
        } catch {
            print("Error signing in: \(error)")
        }
    }
}
```

Add this error alert modifier:

```swift
.alert("Authentication Error", isPresented: $viewModel.showError) {
    Button("OK", role: .cancel) {}
} message: {
    Text(viewModel.errorMessage)
}
```

### Step 4: Reset Password Implementation

Add this code to your settings or profile view:

```swift
// Add this to your view struct
@ObservedObject private var emailViewModel: AuthEmailViewModel = .init()

// Reset password button
Button("Reset Password") {
    Task {
        do {
            try await emailViewModel.resetPassword()
        } catch {
            print("Error resetting password: \(error)")
        }
    }
}
```

## SwiftUI Previews

When creating previews for your views, use:

```swift
struct YourView_Previews: PreviewProvider {
    static var previews: some View {
        YourView(viewModel: AppServices.createAuthViewModel())
    }
}
```
