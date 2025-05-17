![NetSwift Banner](./banner.webp)

# NetSwift

A modern, thread-safe Swift networking library for iOS, macOS, and other Apple platforms, designed with Swift concurrency in mind.

## Features
- ✅ Swift Concurrency built-in with async/await support
- ✅ Strongly typed requests and responses
- ✅ Configurable request building and session management
- ✅ Comprehensive error handling
- ✅ Support for multipart form uploads and raw file uploads
- ✅ Thread-safe token management for authentication
- ✅ Detailed debug logging
- ✅ Easy mocking for unit tests
- ✅ Protocol-oriented design for flexibility and testability

## Installation

### Swift Package Manager

Add NetSwift to your project by adding it as a dependency in your `Package.swift` file:

```swift
dependencies: [
    .package(url: "https://github.com/bibintomj/NetSwift.git", from: "1.0.0")
]
```

## Quick Start

### Basic Usage

```swift
import NetSwift

// 1. Define your API endpoints
enum UserEndpoint: NetworkRequest {
    case getUser(id: Int)
    case updateProfile(User)
    
    var baseURL: URL { URL(string: "https://api.example.com")! }
    
    var path: String {
        switch self {
        case .getUser(let id):
            return "/users/\(id)"
        case .updateProfile:
            return "/users/profile"
        }
    }
    
    var method: HTTPMethod {
        switch self {
        case .getUser: return .get
        case .updateProfile: return .put
        }
    }
    
    var headers: [String: String]? {
        return ["Content-Type": "application/json"]
    }
    
    var queryParameters: [String: String]? {
        return nil
    }
    
    var body: Encodable? {
        switch self {
        case .getUser: return nil
        case .updateProfile(let user): return user
        }
    }
}

// 2. Create a client
let networkClient = NetworkClient()

// 3. Make requests
struct User: Codable {
    var id: Int
    var name: String
    var email: String
}

// GET request
func fetchUser(id: Int) async throws -> User {
    let request = UserEndpoint.getUser(id: id)
    return try await networkClient.request(request)
}

// PUT request
func updateUser(_ user: User) async throws -> User {
    let request = UserEndpoint.updateProfile(user)
    return try await networkClient.request(request)
}
```

## Authentication

NetSwift provides built-in token management with `SessionManager`:

```swift
// After successful login, store tokens
await SessionManager.shared.setTokens(
    accessToken: "your-access-token",
    refreshToken: "your-refresh-token"
)

// Use ProtectedRequestBuilder for authenticated requests
let protectedClient = NetworkClient(
    requestBuilder: ProtectedRequestBuilder()
)

// Tokens will automatically be added to requests
let userProfile: UserProfile = try await protectedClient.request(UserEndpoint.getProfile)

// Clear tokens on logout
await SessionManager.shared.clearTokens()
```

## Uploads

### Raw File Upload

```swift
struct ImageUploadRequest: RawFileUploadRequest {
    let fileData: Data
    
    var baseURL: URL { URL(string: "https://api.example.com")! }
    var path: String { "/uploads/images" }
    var method: HTTPMethod { .post }
    var headers: [String: String]? { ["Content-Type": "image/jpeg"] }
    var queryParameters: [String: String]? { nil }
    var body: Encodable? { nil }
}

// Upload an image file
let imageData = /* your image data */
let request = ImageUploadRequest(fileData: imageData)
let response: UploadResponse = try await networkClient.uploadBinary(request)
```

### Multipart Form Upload

```swift
struct ProfileImageUploadRequest: MultipartUploadRequest {
    let fileData: Data
    let fileName: String
    let mimeType: String
    let formFields: [String: String]
    
    var baseURL: URL { URL(string: "https://api.example.com")! }
    var path: String { "/users/profile-image" }
    var method: HTTPMethod { .post }
    var headers: [String: String]? { nil } // Content-Type will be set automatically
    var queryParameters: [String: String]? { nil }
    var body: Encodable? { nil }
}

// Upload a profile image with additional form fields
let imageData = /* your image data */
let request = ProfileImageUploadRequest(
    fileData: imageData,
    fileName: "profile.jpg",
    mimeType: "image/jpeg",
    formFields: ["userId": "123", "description": "Profile photo"]
)
let response: UploadResponse = try await networkClient.uploadMultipart(request)
```

## Advanced Configuration

### Custom URLSession Configuration

```swift
let configuration = URLSessionConfiguration.default
configuration.timeoutIntervalForRequest = 30
configuration.waitsForConnectivity = true
configuration.httpCookieStorage = nil

let session = URLSession(configuration: configuration)
let networkClient = NetworkClient(session: session)
```

### Custom JSON Encoder/Decoder

```swift
let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
decoder.dateDecodingStrategy = .iso8601

let encoder = JSONEncoder()
encoder.keyEncodingStrategy = .convertToSnakeCase
encoder.dateEncodingStrategy = .iso8601

let networkClient = NetworkClient(decoder: decoder, encoder: encoder)
```

### Enable Debug Logging

```swift
// Enable detailed network logs in debug builds
let networkClient = NetworkClient(debugLoggingEnabled: true)
```

Sample output:
```
🌐⬆️ [NETWORK REQUEST]
🔗 GET https://api.example.com/users/123

📋 HEADERS:
{
  "Content-Type" : "application/json",
  "Authorization" : "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}

🌐⬇️ [NETWORK RESPONSE]
🟢 STATUS: 200
🔗 URL: https://api.example.com/users/123

📦 RESPONSE DATA:
{
  "id" : 123,
  "name" : "John Doe",
  "email" : "john@example.com"
}

🌐✅ [DECODED RESPONSE]
User(id: 123, name: "John Doe", email: "john@example.com")
```

## Error Handling

NetSwift provides comprehensive error types for different failure scenarios:

```swift
do {
    let user: User = try await networkClient.request(UserEndpoint.getUser(id: 123))
    // Handle successful response
} catch let error as NetworkError {
    switch error {
    case .invalidURL:
        // Handle invalid URL
    case .invalidResponse:
        // Handle invalid response format
    case .httpError(let statusCode, let errorBody):
        // Handle HTTP error with structured error response
        print("Server error: \(statusCode), message: \(errorBody?.message ?? "Unknown")")
    case .httpErrorData(let statusCode, let data):
        // Handle HTTP error with raw data
    case .decodingError(let error):
        // Handle JSON decoding error
    case .encodingError(let error):
        // Handle JSON encoding error
    case .unknownError(let error):
        // Handle other errors
    }
} catch {
    // Handle unexpected errors
}
```

## Testing Support

NetSwift's protocol-based design makes it easy to mock for unit tests:

```swift
// Create a mock network session
class MockNetworkSession: NetworkSession {
    var mockData: Data = Data()
    var mockResponse: URLResponse = HTTPURLResponse(url: URL(string: "https://example.com")!, statusCode: 200, httpVersion: nil, headerFields: nil)!
    
    func data(for request: URLRequest) async throws -> (Data, URLResponse) {
        return (mockData, mockResponse)
    }
    
    func upload(for request: URLRequest, from bodyData: Data) async throws -> (Data, URLResponse) {
        return (mockData, mockResponse)
    }
}

// Create a test helper
func createMockNetworkClient(jsonString: String, statusCode: Int = 200) -> NetworkClient {
    let mockSession = MockNetworkSession()
    mockSession.mockData = jsonString.data(using: .utf8)!
    mockSession.mockResponse = HTTPURLResponse(url: URL(string: "https://example.com")!, statusCode: statusCode, httpVersion: nil, headerFields: nil)!
    
    return NetworkClient(session: mockSession)
}

// Use in tests
func testUserFetch() async throws {
    let mockJSON = """
    {
      "id": 123,
      "name": "Test User",
      "email": "test@example.com"
    }
    """
    
    let mockClient = createMockNetworkClient(jsonString: mockJSON)
    let userService = UserService(networkClient: mockClient)
    
    let user = try await userService.fetchUser(id: 123)
    XCTAssertEqual(user.id, 123)
    XCTAssertEqual(user.name, "Test User")
    XCTAssertEqual(user.email, "test@example.com")
}
```

## Requirements

- Swift 5.5+
- iOS 13.0+
- macOS 10.15+
- watchOS 6.0+
- tvOS 13.0+

## License

NetSwift is available under the MIT license. See the LICENSE file for more info.
