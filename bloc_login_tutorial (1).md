# 🔐 ระบบ Login ด้วย BLoC Pattern

## 🎯 เป้าหมาย
สร้างระบบ Login ที่ใช้ BLoC Pattern เชื่อมต่อกับ API และแสดง Token ที่ได้รับ

## 📋 สิ่งที่จะได้เรียนรู้
- การใช้ BLoC กับ API calls
- การจัดการ Loading, Success, Error states
- การใช้ HTTP package
- การส่งข้อมูลจาก Form ไปยัง BLoC

---

## 🛠️ Setup เบื้องต้น

### 1. เพิ่ม Dependencies ใน `pubspec.yaml`

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_bloc: ^8.1.3
  http: ^1.1.0
  equatable: ^2.0.5
  shared_preferences: ^2.2.2

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^2.0.0
```

**คำอธิบาย Dependencies:**
- `flutter_bloc`: สำหรับ BLoC state management
- `http`: สำหรับเรียก API
- `equatable`: สำหรับเปรียบเทียบ objects ใน BLoC
- `shared_preferences`: สำหรับบันทึก token ใน local storage

### 2. โครงสร้างไฟล์ (Clean Architecture)

```
lib/
├── main.dart
├── data/
│   ├── models/
│   │   ├── login_request.dart
│   │   └── login_response.dart
│   ├── repositories/
│   │   └── auth_repository_impl.dart
│   └── services/
│       └── auth_api_service.dart
├── domain/
│   ├── entities/
│   │   ├── user.dart
│   │   └── auth_token.dart
│   └── repositories/
│       └── auth_repository.dart
├── presentation/
│   ├── bloc/
│   │   ├── login_event.dart
│   │   ├── login_state.dart
│   │   └── login_bloc.dart
│   └── screens/
│       └── login_screen.dart
└── core/
    └── constants/
        └── api_constants.dart
```

---

## 🏛️ Clean Architecture Concepts

### Model vs Entity vs Repository

#### 📊 **Entity (Domain Layer)**
- **ความหมาย:** โครงสร้างข้อมูลหลักของ Business Logic
- **จุดประสงค์:** แทน concept ที่สำคัญในระบบ (User, Product, Order)
- **ลักษณะ:** ไม่ขึ้นกับ external system, มี business rules

#### 📋 **Model (Data Layer)**  
- **ความหมาย:** โครงสร้างข้อมูลสำหรับรับส่งข้อมูลกับ external system
- **จุดประสงค์:** แปลงข้อมูลจาก/ไป API, Database
- **ลักษณะ:** มี `fromJson()`, `toJson()`, field ตรงกับ API

#### 🏪 **Repository Pattern**
- **ความหมาย:** Interface ระหว่าง Business Logic กับ Data Source
- **จุดประสงค์:** ซ่อน implementation details ของการดึงข้อมูล
- **ประโยชน์:** 
  - เปลี่ยน data source ได้ง่าย (API → Database)
  - Testing ง่ายขึ้น (mock repository)
  - Code ที่ clean และ maintainable

### การทำงานของ Clean Architecture

```
UI → BLoC → Repository → API Service → API
                ↓
              Entity ← Model (fromJson)
```

---

## 🏗️ Core Constants

```dart
// lib/core/constants/api_constants.dart
class ApiConstants {
  static const String baseUrl = 'https://api.dev.dedepos.com';
  static const String loginEndpoint = '/login';
  
  // Headers
  static const Map<String, String> defaultHeaders = {
    'accept': 'application/json',
    'Content-Type': 'application/json',
  };
  
  // Timeouts
  static const Duration connectionTimeout = Duration(seconds: 30);
  static const Duration receiveTimeout = Duration(seconds: 30);
}
```

---

## 🌐 Domain Layer

### 1. สร้าง Entities

```dart
// lib/domain/entities/auth_token.dart
class AuthToken {
  final String accessToken;
  final String refreshToken;
  final DateTime? expiresAt;

  const AuthToken({
    required this.accessToken,
    required this.refreshToken,
    this.expiresAt,
  });

  bool get isExpired {
    if (expiresAt == null) return false;
    return DateTime.now().isAfter(expiresAt!);
  }

  @override
  String toString() {
    return 'AuthToken{accessToken: ${accessToken.substring(0, 10)}..., refreshToken: ${refreshToken.substring(0, 10)}...}';
  }
}
```

```dart
// lib/domain/entities/user.dart
class User {
  final String id;
  final String username;
  final String? email;
  final String? firstName;
  final String? lastName;

  const User({
    required this.id,
    required this.username,
    this.email,
    this.firstName,
    this.lastName,
  });

  String get fullName {
    if (firstName != null && lastName != null) {
      return '$firstName $lastName';
    }
    return username;
  }

  @override
  String toString() {
    return 'User{id: $id, username: $username, email: $email}';
  }
}
```

### 2. สร้าง Repository Interface

```dart
// lib/domain/repositories/auth_repository.dart
import '../entities/auth_token.dart';
import '../entities/user.dart';

abstract class AuthRepository {
  Future<AuthToken> login({
    required String username,
    required String password,
  });
  
  Future<void> logout();
  
  Future<AuthToken?> getStoredToken();
  
  Future<void> saveToken(AuthToken token);
  
  Future<void> clearToken();
  
  Future<AuthToken> refreshToken(String refreshToken);
  
  Future<User?> getCurrentUser();
}
```

---

## 📊 Data Layer

### 1. สร้าง Models สำหรับ API

```dart
// lib/data/models/login_request.dart
class LoginRequest {
  final String username;
  final String password;

  const LoginRequest({
    required this.username,
    required this.password,
  });

  Map<String, dynamic> toJson() {
    return {
      'username': username,
      'password': password,
    };
  }

  @override
  String toString() {
    return 'LoginRequest{username: $username, password: ***}';
  }
}
```

```dart
// lib/data/models/login_response.dart
import '../../domain/entities/auth_token.dart';

class LoginResponse {
  final String refresh;
  final bool success;
  final String token;

  const LoginResponse({
    required this.refresh,
    required this.success,
    required this.token,
  });

  factory LoginResponse.fromJson(Map<String, dynamic> json) {
    return LoginResponse(
      refresh: json['refresh'] ?? '',
      success: json['success'] ?? false,
      token: json['token'] ?? '',
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'refresh': refresh,
      'success': success,
      'token': token,
    };
  }

  // แปลง Model เป็น Entity
  AuthToken toEntity() {
    return AuthToken(
      accessToken: token,
      refreshToken: refresh,
      // สามารถเพิ่ม expiresAt ได้ถ้า API ส่งมา
    );
  }

  @override
  String toString() {
    return 'LoginResponse{refresh: ${refresh.substring(0, 10)}..., success: $success, token: ${token.substring(0, 10)}...}';
  }
}
```

### 2. สร้าง API Service

```dart
// lib/data/services/auth_api_service.dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import '../models/login_request.dart';
import '../models/login_response.dart';
import '../../core/constants/api_constants.dart';

class AuthApiService {
  Future<LoginResponse> login(LoginRequest request) async {
    try {
      final url = Uri.parse('${ApiConstants.baseUrl}${ApiConstants.loginEndpoint}');
      
      print('🌐 API Call: POST $url');
      print('📤 Request body: ${jsonEncode(request.toJson())}');
      
      final response = await http.post(
        url,
        headers: ApiConstants.defaultHeaders,
        body: jsonEncode(request.toJson()),
      ).timeout(ApiConstants.connectionTimeout);

      print('📊 Response status: ${response.statusCode}');
      print('📥 Response body: ${response.body}');

      if (response.statusCode == 200) {
        final jsonData = jsonDecode(response.body);
        return LoginResponse.fromJson(jsonData);
      } else if (response.statusCode == 401) {
        throw UnauthorizedException('Invalid username or password');
      } else if (response.statusCode >= 500) {
        throw ServerException('Server error: ${response.statusCode}');
      } else {
        throw ApiException('Request failed: ${response.statusCode}');
      }
    } on http.ClientException catch (e) {
      throw NetworkException('Network error: ${e.message}');
    } catch (e) {
      if (e is AuthException) rethrow;
      throw UnknownException('Unexpected error: $e');
    }
  }
}

// Custom Exceptions
abstract class AuthException implements Exception {
  final String message;
  const AuthException(this.message);
  
  @override
  String toString() => message;
}

class NetworkException extends AuthException {
  const NetworkException(String message) : super(message);
}

class UnauthorizedException extends AuthException {
  const UnauthorizedException(String message) : super(message);
}

class ServerException extends AuthException {
  const ServerException(String message) : super(message);
}

class ApiException extends AuthException {
  const ApiException(String message) : super(message);
}

class UnknownException extends AuthException {
  const UnknownException(String message) : super(message);
}
```

### 3. สร้าง Repository Implementation

```dart
// lib/data/repositories/auth_repository_impl.dart
import 'package:shared_preferences/shared_preferences.dart';
import '../../domain/entities/auth_token.dart';
import '../../domain/entities/user.dart';
import '../../domain/repositories/auth_repository.dart';
import '../models/login_request.dart';
import '../services/auth_api_service.dart';

class AuthRepositoryImpl implements AuthRepository {
  final AuthApiService _apiService;
  
  AuthRepositoryImpl(this._apiService);

  @override
  Future<AuthToken> login({
    required String username,
    required String password,
  }) async {
    try {
      final request = LoginRequest(
        username: username,
        password: password,
      );
      
      final response = await _apiService.login(request);
      
      if (!response.success) {
        throw UnauthorizedException('Login failed: Invalid credentials');
      }
      
      final authToken = response.toEntity();
      
      // บันทึก token ลง local storage
      await saveToken(authToken);
      
      return authToken;
    } catch (e) {
      print('❌ Login repository error: $e');
      rethrow;
    }
  }

  @override
  Future<void> logout() async {
    await clearToken();
  }

  @override
  Future<AuthToken?> getStoredToken() async {
    try {
      final prefs = await SharedPreferences.getInstance();
      final accessToken = prefs.getString('access_token');
      final refreshToken = prefs.getString('refresh_token');
      final expiresAtString = prefs.getString('expires_at');
      
      if (accessToken == null || refreshToken == null) {
        return null;
      }
      
      DateTime? expiresAt;
      if (expiresAtString != null) {
        expiresAt = DateTime.parse(expiresAtString);
      }
      
      return AuthToken(
        accessToken: accessToken,
        refreshToken: refreshToken,
        expiresAt: expiresAt,
      );
    } catch (e) {
      print('❌ Get stored token error: $e');
      return null;
    }
  }

  @override
  Future<void> saveToken(AuthToken token) async {
    try {
      final prefs = await SharedPreferences.getInstance();
      await prefs.setString('access_token', token.accessToken);
      await prefs.setString('refresh_token', token.refreshToken);
      
      if (token.expiresAt != null) {
        await prefs.setString('expires_at', token.expiresAt!.toIso8601String());
      }
      
      print('✅ Token saved successfully');
    } catch (e) {
      print('❌ Save token error: $e');
    }
  }

  @override
  Future<void> clearToken() async {
    try {
      final prefs = await SharedPreferences.getInstance();
      await prefs.remove('access_token');
      await prefs.remove('refresh_token');
      await prefs.remove('expires_at');
      
      print('✅ Token cleared successfully');
    } catch (e) {
      print('❌ Clear token error: $e');
    }
  }

  @override
  Future<AuthToken> refreshToken(String refreshToken) async {
    // TODO: Implement refresh token logic
    throw UnimplementedError('Refresh token not implemented yet');
  }

  @override
  Future<User?> getCurrentUser() async {
    // TODO: Implement get current user logic
    throw UnimplementedError('Get current user not implemented yet');
  }
}

---

## 🎬 Presentation Layer

### 1. สร้าง Events

```dart
// lib/presentation/bloc/login_event.dart
import 'package:equatable/equatable.dart';

abstract class LoginEvent extends Equatable {
  const LoginEvent();

  @override
  List<Object> get props => [];
}

// เหตุการณ์เมื่อผู้ใช้กดปุ่ม Login
class LoginSubmitted extends LoginEvent {
  final String username;
  final String password;

  const LoginSubmitted({
    required this.username,
    required this.password,
  });

  @override
  List<Object> get props => [username, password];
}

// เหตุการณ์เมื่อต้องการ Reset state
class LoginReset extends LoginEvent {}

// เหตุการณ์เมื่อต้องการตรวจสอบ Token ที่เก็บไว้
class LoginCheckAuthStatus extends LoginEvent {}
```

### 2. สร้าง States

```dart
// lib/presentation/bloc/login_state.dart
import 'package:equatable/equatable.dart';
import '../../domain/entities/auth_token.dart';

abstract class LoginState extends Equatable {
  const LoginState();

  @override
  List<Object?> get props => [];
}

// สถานะเริ่มต้น
class LoginInitial extends LoginState {}

// สถานะกำลังโหลด
class LoginLoading extends LoginState {}

// สถานะสำเร็จ
class LoginSuccess extends LoginState {
  final AuthToken authToken;

  const LoginSuccess(this.authToken);

  @override
  List<Object> get props => [authToken];
}

// สถานะเกิดข้อผิดพลาด
class LoginError extends LoginState {
  final String message;

  const LoginError(this.message);

  @override
  List<Object> get props => [message];
}

// สถานะเมื่อมี Token อยู่แล้ว
class LoginAuthenticated extends LoginState {
  final AuthToken authToken;

  const LoginAuthenticated(this.authToken);

  @override
  List<Object> get props => [authToken];
}
```

### 3. สร้าง BLoC

```dart
// lib/presentation/bloc/login_bloc.dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'login_event.dart';
import 'login_state.dart';
import '../../domain/repositories/auth_repository.dart';
import '../../data/services/auth_api_service.dart';

class LoginBloc extends Bloc<LoginEvent, LoginState> {
  final AuthRepository _authRepository;

  LoginBloc(this._authRepository) : super(LoginInitial()) {
    // จัดการเหตุการณ์ LoginSubmitted
    on<LoginSubmitted>(_onLoginSubmitted);
    
    // จัดการเหตุการณ์ LoginReset
    on<LoginReset>(_onLoginReset);
    
    // จัดการเหตุการณ์ LoginCheckAuthStatus
    on<LoginCheckAuthStatus>(_onLoginCheckAuthStatus);
  }

  Future<void> _onLoginSubmitted(
    LoginSubmitted event,
    Emitter<LoginState> emit,
  ) async {
    emit(LoginLoading());

    try {
      final authToken = await _authRepository.login(
        username: event.username,
        password: event.password,
      );

      emit(LoginSuccess(authToken));
    } on UnauthorizedException catch (e) {
      emit(LoginError('ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง'));
    } on NetworkException catch (e) {
      emit(LoginError('ไม่สามารถเชื่อมต่อเครือข่ายได้ กรุณาลองใหม่อีกครั้ง'));
    } on ServerException catch (e) {
      emit(LoginError('เซิร์ฟเวอร์ขัดข้อง กรุณาลองใหม่ภายหลัง'));
    } catch (e) {
      emit(LoginError('เกิดข้อผิดพลาดที่ไม่คาดคิด: ${e.toString()}'));
    }
  }

  void _onLoginReset(
    LoginReset event,
    Emitter<LoginState> emit,
  ) {
    emit(LoginInitial());
  }

  Future<void> _onLoginCheckAuthStatus(
    LoginCheckAuthStatus event,
    Emitter<LoginState> emit,
  ) async {
    try {
      final storedToken = await _authRepository.getStoredToken();
      
      if (storedToken != null && !storedToken.isExpired) {
        emit(LoginAuthenticated(storedToken));
      } else {
        // Token หมดอายุหรือไม่มี ให้ clear ออก
        await _authRepository.clearToken();
        emit(LoginInitial());
      }
    } catch (e) {
      emit(LoginInitial());
    }
  }
}
```

---

## 🖼️ 4. สร้าง UI (Login Screen)

```dart
// lib/presentation/screens/login_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import '../bloc/login_bloc.dart';
import '../bloc/login_event.dart';
import '../bloc/login_state.dart';
import '../../data/services/auth_api_service.dart';
import '../../data/repositories/auth_repository_impl.dart';

class LoginScreen extends StatefulWidget {
  const LoginScreen({super.key});

  @override
  State<LoginScreen> createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final _formKey = GlobalKey<FormState>();
  final _usernameController = TextEditingController(text: 'maxkorn'); // Pre-fill for testing
  final _passwordController = TextEditingController(text: 'maxkorn'); // Pre-fill for testing

  @override
  void dispose() {
    _usernameController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Login with BLoC + Repository'),
        backgroundColor: Colors.blue,
        foregroundColor: Colors.white,
      ),
      body: BlocProvider(
        // Dependency Injection - สร้าง BLoC พร้อม Repository
        create: (context) => LoginBloc(
          AuthRepositoryImpl(AuthApiService()),
        )..add(LoginCheckAuthStatus()), // ตรวจสอบ token ที่เก็บไว้
        child: BlocConsumer<LoginBloc, LoginState>(
          listener: (context, state) {
            if (state is LoginError) {
              ScaffoldMessenger.of(context).showSnackBar(
                SnackBar(
                  content: Text(state.message),
                  backgroundColor: Colors.red,
                  action: SnackBarAction(
                    label: 'ปิด',
                    textColor: Colors.white,
                    onPressed: () {
                      ScaffoldMessenger.of(context).hideCurrentSnackBar();
                    },
                  ),
                ),
              );
            } else if (state is LoginSuccess) {
              ScaffoldMessenger.of(context).showSnackBar(
                const SnackBar(
                  content: Text('เข้าสู่ระบบสำเร็จ!'),
                  backgroundColor: Colors.green,
                ),
              );
            }
          },
          builder: (context, state) {
            // แสดงหน้า Token ถ้ามี token อยู่แล้ว
            if (state is LoginAuthenticated || state is LoginSuccess) {
              final token = state is LoginAuthenticated 
                  ? state.authToken 
                  : (state as LoginSuccess).authToken;
              return _buildTokenDisplay(context, token);
            }

            return Padding(
              padding: const EdgeInsets.all(16.0),
              child: Form(
                key: _formKey,
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    // Header
                    const Icon(
                      Icons.lock_outline,
                      size: 80,
                      color: Colors.blue,
                    ),
                    const SizedBox(height: 24),
                    const Text(
                      'เข้าสู่ระบบ',
                      style: TextStyle(
                        fontSize: 28,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                    const SizedBox(height: 8),
                    Text(
                      'Clean Architecture + BLoC Pattern',
                      style: TextStyle(
                        fontSize: 14,
                        color: Colors.grey[600],
                      ),
                    ),
                    const SizedBox(height: 32),

                    // Username Field
                    TextFormField(
                      controller: _usernameController,
                      decoration: const InputDecoration(
                        labelText: 'Username',
                        border: OutlineInputBorder(),
                        prefixIcon: Icon(Icons.person),
                        helperText: 'ใช้ maxkorn สำหรับทดสอบ',
                      ),
                      validator: (value) {
                        if (value == null || value.isEmpty) {
                          return 'กรุณาใส่ username';
                        }
                        return null;
                      },
                    ),
                    const SizedBox(height: 16),

                    // Password Field
                    TextFormField(
                      controller: _passwordController,
                      obscureText: true,
                      decoration: const InputDecoration(
                        labelText: 'Password',
                        border: OutlineInputBorder(),
                        prefixIcon: Icon(Icons.lock),
                        helperText: 'ใช้ maxkorn สำหรับทดสอบ',
                      ),
                      validator: (value) {
                        if (value == null || value.isEmpty) {
                          return 'กรุณาใส่ password';
                        }
                        return null;
                      },
                    ),
                    const SizedBox(height: 24),

                    // Login Button
                    SizedBox(
                      width: double.infinity,
                      height: 50,
                      child: ElevatedButton(
                        onPressed: state is LoginLoading
                            ? null
                            : () => _submitLogin(context),
                        style: ElevatedButton.styleFrom(
                          backgroundColor: Colors.blue,
                          foregroundColor: Colors.white,
                          shape: RoundedRectangleBorder(
                            borderRadius: BorderRadius.circular(8),
                          ),
                        ),
                        child: state is LoginLoading
                            ? const SizedBox(
                                width: 20,
                                height: 20,
                                child: CircularProgressIndicator(
                                  color: Colors.white,
                                  strokeWidth: 2,
                                ),
                              )
                            : const Text(
                                'เข้าสู่ระบบ',
                                style: TextStyle(fontSize: 18),
                              ),
                      ),
                    ),
                    const SizedBox(height: 16),

                    // Reset Button
                    if (state is LoginError)
                      TextButton(
                        onPressed: () {
                          context.read<LoginBloc>().add(LoginReset());
                        },
                        child: const Text('ลองใหม่อีกครั้ง'),
                      ),
                  ],
                ),
              ),
            );
          },
        ),
      ),
    );
  }

  Widget _buildTokenDisplay(BuildContext context, authToken) {
    return Padding(
      padding: const EdgeInsets.all(16.0),
      child: Column(
        children: [
          // Header
          Container(
            width: double.infinity,
            padding: const EdgeInsets.all(20),
            decoration: BoxDecoration(
              color: Colors.green[50],
              borderRadius: BorderRadius.circular(12),
              border: Border.all(color: Colors.green[200]!),
            ),
            child: Column(
              children: [
                const Icon(
                  Icons.check_circle,
                  color: Colors.green,
                  size: 60,
                ),
                const SizedBox(height: 12),
                const Text(
                  'เข้าสู่ระบบสำเร็จ!',
                  style: TextStyle(
                    color: Colors.green,
                    fontWeight: FontWeight.bold,
                    fontSize: 20,
                  ),
                ),
                const SizedBox(height: 8),
                Text(
                  'Token ถูกบันทึกใน Local Storage แล้ว',
                  style: TextStyle(
                    color: Colors.green[700],
                    fontSize: 14,
                  ),
                ),
              ],
            ),
          ),
          
          const SizedBox(height: 24),

          Expanded(
            child: SingleChildScrollView(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  // Access Token
                  _buildTokenCard(
                    title: 'Access Token',
                    token: authToken.accessToken,
                    icon: Icons.key,
                    color: Colors.blue,
                  ),
                  
                  const SizedBox(height: 16),
                  
                  // Refresh Token
                  _buildTokenCard(
                    title: 'Refresh Token',
                    token: authToken.refreshToken,
                    icon: Icons.refresh,
                    color: Colors.orange,
                  ),
                  
                  const SizedBox(height: 16),
                  
                  // Token Info
                  Card(
                    child: Padding(
                      padding: const EdgeInsets.all(16),
                      child: Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          Row(
                            children: [
                              Icon(Icons.info, color: Colors.grey[600]),
                              const SizedBox(width: 8),
                              const Text(
                                'Token Information',
                                style: TextStyle(fontWeight: FontWeight.bold),
                              ),
                            ],
                          ),
                          const SizedBox(height: 12),
                          _buildInfoRow('สถานะ', 'Active'),
                          _buildInfoRow('Type', 'Bearer Token'),
                          _buildInfoRow('เก็บไว้ใน', 'SharedPreferences'),
                          if (authToken.expiresAt != null)
                            _buildInfoRow('หมดอายุ', authToken.expiresAt.toString()),
                        ],
                      ),
                    ),
                  ),
                ],
              ),
            ),
          ),
          
          const SizedBox(height: 16),
          
          // Action Buttons
          Column(
            children: [
              SizedBox(
                width: double.infinity,
                child: ElevatedButton.icon(
                  onPressed: () {
                    context.read<LoginBloc>().add(LoginReset());
                  },
                  icon: const Icon(Icons.login),
                  label: const Text('เข้าสู่ระบบใหม่'),
                  style: ElevatedButton.styleFrom(
                    backgroundColor: Colors.blue,
                    foregroundColor: Colors.white,
                  ),
                ),
              ),
              const SizedBox(height: 8),
              SizedBox(
                width: double.infinity,
                child: OutlinedButton.icon(
                  onPressed: () async {
                    // ลบ token และ reset state
                    final bloc = context.read<LoginBloc>();
                    bloc.add(LoginReset());
                    
                    // แสดงข้อความยืนยัน
                    ScaffoldMessenger.of(context).showSnackBar(
                      const SnackBar(
                        content: Text('ออกจากระบบแล้ว'),
                        backgroundColor: Colors.orange,
                      ),
                    );
                  },
                  icon: const Icon(Icons.logout),
                  label: const Text('ออกจากระบบ'),
                  style: OutlinedButton.styleFrom(
                    foregroundColor: Colors.red,
                  ),
                ),
              ),
            ],
          ),
        ],
      ),
    );
  }

  Widget _buildTokenCard({
    required String title,
    required String token,
    required IconData icon,
    required Color color,
  }) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                Icon(icon, color: color),
                const SizedBox(width: 8),
                Text(
                  title,
                  style: const TextStyle(fontWeight: FontWeight.bold),
                ),
                const Spacer(),
                IconButton(
                  icon: const Icon(Icons.copy, size: 20),
                  onPressed: () {
                    // TODO: Copy to clipboard
                    ScaffoldMessenger.of(context).showSnackBar(
                      SnackBar(content: Text('คัดลอก $title แล้ว')),
                    );
                  },
                ),
              ],
            ),
            const SizedBox(height: 8),
            Container(
              width: double.infinity,
              padding: const EdgeInsets.all(12),
              decoration: BoxDecoration(
                color: Colors.grey[100],
                borderRadius: BorderRadius.circular(8),
                border: Border.all(color: Colors.grey[300]!),
              ),
              child: SelectableText(
                token,
                style: const TextStyle(
                  fontFamily: 'monospace',
                  fontSize: 12,
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildInfoRow(String label, String value) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          SizedBox(
            width: 80,
            child: Text(
              '$label:',
              style: const TextStyle(fontWeight: FontWeight.w500),
            ),
          ),
          Expanded(
            child: Text(value),
          ),
        ],
      ),
    );
  }

  void _submitLogin(BuildContext context) {
    if (_formKey.currentState!.validate()) {
      context.read<LoginBloc>().add(
            LoginSubmitted(
              username: _usernameController.text.trim(),
              password: _passwordController.text.trim(),
            ),
          );
    }
  }
}
```

---

## 🏁 5. Main App

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'presentation/screens/login_screen.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'BLoC + Repository Login Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
        useMaterial3: true,
        appBarTheme: const AppBarTheme(
          centerTitle: true,
          elevation: 2,
        ),
      ),
      home: const LoginScreen(),
      debugShowCheckedModeBanner: false,
    );
  }
}
```

---

## 🧪 6. การทดสอบ

### ข้อมูลสำหรับทดสอบ:
- **Username:** `maxkorn`
- **Password:** `maxkorn`

### ผลลัพธ์ที่ควรได้:
```json
{
  "refresh": "c31c2b7b23385510c7336c82b1c6d4790b59f858618ae8b7853c0918262ca125",
  "success": true,
  "token": "18370fdff14e3fb03a00c3523b25e66b92aaf109ccc64cc43180247389a40945"
}
```

### Scenarios การทดสอบ:
1. **Login สำเร็จ:** ใส่ username/password ที่ถูกต้อง
2. **Login ไม่สำเร็จ:** ใส่ข้อมูลผิด
3. **Network Error:** ปิด internet แล้วลอง login
4. **Token Persistence:** ปิดแอปแล้วเปิดใหม่ (token ควรยังอยู่)
5. **Logout:** กดปุ่มออกจากระบบ

---

## 🏗️ Architecture Overview

### Clean Architecture Layers:

```
┌─────────────────────────────────────┐
│          Presentation Layer         │
│  (UI, BLoC, Events, States)        │
├─────────────────────────────────────┤
│           Domain Layer              │
│  (Entities, Repository Interfaces)  │
├─────────────────────────────────────┤
│            Data Layer               │
│  (Models, Repository Impl, API)     │
└─────────────────────────────────────┘
```

### Data Flow:

```
UI → Event → BLoC → Repository → API Service → API
                        ↓
    UI ← State ← BLoC ← Entity ← Model (fromJson)
```

### Benefits ของ Clean Architecture:

1. **Separation of Concerns:** แต่ละ layer มีหน้าที่ชัดเจน
2. **Testability:** สามารถ mock แต่ละ layer ได้
3. **Maintainability:** เปลี่ยนแปลงในส่วนหนึ่งไม่กระทบส่วนอื่น
4. **Scalability:** เพิ่มฟีเจอร์ใหม่ได้ง่าย

---

## 🔍 อธิบาย Flow การทำงาน

### 1. App Start
```
main() → MyApp → LoginScreen → BLoC creation → LoginCheckAuthStatus
```

### 2. Login Process  
```
User Input → Form Validation → LoginSubmitted Event → BLoC
           → Repository → API Service → HTTP Request → API
           → Response → Model → Entity → State → UI Update
```

### 3. Token Management
```
Login Success → AuthToken Entity → Repository.saveToken() 
              → SharedPreferences → Local Storage
```

### 4. App Restart
```
App Start → LoginCheckAuthStatus → Repository.getStoredToken()
          → SharedPreferences → Check Expiry → LoginAuthenticated State
```

---

## 💡 จุดสำคัญที่ต้องเข้าใจ

### 1. **Clean Architecture Benefits**
- **Domain Layer:** Business logic ที่ไม่ขึ้นกับ framework
- **Data Layer:** การจัดการข้อมูลจาก external sources  
- **Presentation Layer:** UI และ state management

### 2. **Repository Pattern**
- **Interface:** อยู่ใน Domain layer (abstraction)
- **Implementation:** อยู่ใน Data layer (concrete)
- **Benefits:** Dependency Inversion, Easy testing, Flexible data sources

### 3. **Entity vs Model**
- **Entity:** Business object ที่ clean, ไม่ขึ้นกับ external system
- **Model:** Data transfer object, มี toJson/fromJson

### 4. **Dependency Injection**
- BLoC รับ Repository ผ่าน constructor
- Repository รับ API Service ผ่าน constructor
- ทำให้ testing และ mocking ง่าย

### 5. **Error Handling**
- Custom exceptions สำหรับแต่ละประเภทของ error
- User-friendly error messages
- Graceful degradation

---

## 📝 แบบฝึกหัดเพิ่มเติม

### Level 1: พื้นฐาน
1. **เพิ่ม Validation:** 
   - Email format validation
   - Password strength indicator
   - Show validation errors แบบ real-time

2. **UI Improvements:**
   - Remember me checkbox
   - Show/hide password toggle
   - Loading animation ที่สวยกว่า

### Level 2: กลาง  
3. **Token Management:**
   - Auto refresh token เมื่อใกล้หมดอายุ
   - Handle token expiry gracefully
   - Multiple login sessions

4. **Navigation:**
   - เพิ่มหน้า Home หลัง login สำเร็จ
   - Protected routes ที่ต้อง login ก่อน
   - Deep linking support

### Level 3: ขั้นสูง
5. **Advanced Features:**
   - Biometric authentication
   - Multi-factor authentication (2FA)
   - Social login (Google, Facebook)

6. **Architecture Improvements:**
   - Dependency injection framework (get_it)
   - Event sourcing สำหรับ audit log
   - Offline-first architecture

---

## 🎯 เป้าหมายการเรียนรู้

หลังจากทำแบบฝึกหัดนี้ นักศึกษาควรเข้าใจ:

### ✅ Technical Skills
- Clean Architecture principles
- Repository Pattern implementation  
- BLoC state management ขั้นสูง
- Error handling strategies
- Token-based authentication

### ✅ Best Practices
- Separation of concerns
- Dependency injection
- Custom exception handling
- Proper state management
- Local data persistence

### ✅ Professional Development
- Code organization และ structure
- Testing considerations
- Maintenance และ scalability
- Security best practices

---

## 🚀 ขั้นตอนถัดไป

### สัปดาห์ถัดไป:
1. **User Management:** Profile screen, User preferences
2. **Data Persistence:** SQLite integration with Repository
3. **Advanced State Management:** MultiBLoC coordination
4. **API Integration:** CRUD operations with backend
5. **Testing:** Unit tests, Widget tests, Integration tests

### โปรเจคขั้นสูง:
- **E-commerce App:** Product catalog, Shopping cart, Order management
- **Social Media App:** Posts, Comments, Real-time chat
- **Finance App:** Transactions, Budgeting, Reports