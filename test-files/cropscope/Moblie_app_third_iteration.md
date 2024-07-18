Sure! Here are the exact steps to follow to get to the end goal using the provided code. This guide is aimed at beginners who are familiar with programming but new to Flutter.

### Step 1: Set Up Your Development Environment

1. **Install Flutter and Dart:**
   
   - Follow the official installation guide: [Flutter Installation](https://flutter.dev/docs/get-started/install)

2. **Set Up an IDE:**
   
   - Install Visual Studio Code (VS Code) or Android Studio.
   - Install Flutter and Dart plugins for your chosen IDE.

### Step 2: Create a New Flutter Project

1. **Open Terminal or Command Prompt:**
   
   ```sh
   flutter create crop_scope
   cd crop_scope
   ```

2. **Open the Project in Your IDE:**
   
   - If using VS Code: `code .`

### Step 3: Organize Your Project Structure

1. **Create Directories:**
   
   - In the `lib` directory, create folders for `pages`, `services`, and `models`:
     
     ```sh
     mkdir lib/pages lib/services lib/models
     ```

### Step 4: Add Dependencies

1. **Open `pubspec.yaml`:**
   
   - Add the necessary dependencies:
     
     ```yaml
     dependencies:
       flutter:
         sdk: flutter
       provider: ^6.0.0
       http: ^0.13.3
       flutter_secure_storage: ^5.0.2
     ```

2. **Run `flutter pub get`** to install dependencies:
   
   ```sh
   flutter pub get
   ```

### Step 5: Implement the Authentication Service

1. **Create `auth_service.dart` in `lib/services`:**
   
   ```dart
   import 'dart:convert';
   import 'package:flutter_secure_storage/flutter_secure_storage.dart';
   import 'package:http/http.dart' as http;
   
   class AuthService {
     final String baseUrl;
     final FlutterSecureStorage storage = FlutterSecureStorage();
   
     AuthService({required this.baseUrl});
   
     Future<bool> login(String email, String password) async {
       final response = await http.post(
         Uri.parse('$baseUrl/login'),
         headers: {'Content-Type': 'application/json'},
         body: jsonEncode({'email': email, 'password': password}),
       );
   
       if (response.statusCode == 200) {
         final data = jsonDecode(response.body);
         await storage.write(key: 'token', value: data['token']);
         return true;
       } else {
         return false;
       }
     }
   
     Future<void> logout() async {
       await storage.delete(key: 'token');
     }
   
     Future<String?> getToken() async {
       return await storage.read(key: 'token');
     }
   }
   ```

### Step 6: Implement the API Service

1. **Create `api_service.dart` in `lib/services`:**
   
   ```dart
   import 'dart:convert';
   import 'package:http/http.dart' as http;
   import 'auth_service.dart';
   
   class ApiService {
     final String baseUrl;
     final AuthService authService;
   
     ApiService({required this.baseUrl, required this.authService});
   
     Future<Map<String, dynamic>> fetchData(String endpoint) async {
       final token = await authService.getToken();
       final response = await http.get(
         Uri.parse('$baseUrl/$endpoint'),
         headers: {'Authorization': 'Bearer $token'},
       );
   
       if (response.statusCode == 200) {
         return jsonDecode(response.body);
       } else {
         throw Exception('Failed to load data');
       }
     }
   }
   ```

### Step 7: Implement Login Page

1. **Create `login_page.dart` in `lib/pages`:**
   
   ```dart
   import 'package:flutter/material.dart';
   import 'package:provider/provider.dart';
   import '../services/auth_service.dart';
   import 'dashboard.dart';
   import 'login_failed_page.dart';
   
   class LoginPage extends StatefulWidget {
     @override
     _LoginPageState createState() => _LoginPageState();
   }
   
   class _LoginPageState extends State<LoginPage> {
     final TextEditingController emailController = TextEditingController();
     final TextEditingController passwordController = TextEditingController();
     bool _isLoading = false;
   
     @override
     Widget build(BuildContext context) {
       return Scaffold(
         body: Padding(
           padding: const EdgeInsets.all(16.0),
           child: Column(
             mainAxisAlignment: MainAxisAlignment.center,
             children: <Widget>[
               Text("CropScope", style: TextStyle(fontSize: 32, fontWeight: FontWeight.bold)),
               TextField(
                 controller: emailController,
                 decoration: InputDecoration(labelText: 'Email or Mobile number'),
               ),
               TextField(
                 controller: passwordController,
                 decoration: InputDecoration(labelText: 'Password'),
                 obscureText: true,
               ),
               SizedBox(height: 20),
               _isLoading
                   ? CircularProgressIndicator()
                   : ElevatedButton(
                       onPressed: () async {
                         setState(() {
                           _isLoading = true;
                         });
                         final email = emailController.text;
                         final password = passwordController.text;
                         bool success = await Provider.of<AuthService>(context, listen: false)
                             .login(email, password);
                         setState(() {
                           _isLoading = false;
                         });
                         if (success) {
                           Navigator.pushReplacement(context, MaterialPageRoute(builder: (context) => DashboardPage()));
                         } else {
                           Navigator.pushReplacement(context, MaterialPageRoute(builder: (context) => LoginFailedPage()));
                         }
                       },
                       child: Text("Log in"),
                     ),
               TextButton(
                 onPressed: () {
                   // Implement forgot password functionality
                 },
                 child: Text("Forgot password?"),
               ),
             ],
           ),
         ),
       );
     }
   }
   ```

### Step 8: Implement Login Failed Page

1. **Create `login_failed_page.dart` in `lib/pages`:**
   
   ```dart
   import 'package:flutter/material.dart';
   import 'login_page.dart';
   
   class LoginFailedPage extends StatelessWidget {
     @override
     Widget build(BuildContext context) {
       return Scaffold(
         appBar: AppBar(
           title: Text("Login Failed"),
         ),
         body: Center(
           child: Column(
             mainAxisAlignment: MainAxisAlignment.center,
             children: <Widget>[
               Text("Login was unsuccessful.", style: TextStyle(fontSize: 24)),
               SizedBox(height: 20),
               ElevatedButton(
                 onPressed: () {
                   Navigator.pushReplacement(context, MaterialPageRoute(builder: (context) => LoginPage()));
                 },
                 child: Text("Back to Login"),
               ),
             ],
           ),
         ),
       );
     }
   }
   ```

### Step 9: Implement Dashboard Page

1. **Create `dashboard.dart` in `lib/pages`:**
   
   ```dart
   import 'package:flutter/material.dart';
   import 'package:provider/provider.dart';
   import '../services/auth_service.dart';
   import '../services/api_service.dart';
   import 'recommendations.dart';
   import 'login_page.dart';
   
   class DashboardPage extends StatefulWidget {
     @override
     _DashboardPageState createState() => _DashboardPageState();
   }
   
   class _DashboardPageState extends State<DashboardPage> {
     late Future<Map<String, dynamic>> _data;
   
     @override
     void initState() {
       super.initState();
       final authService = Provider.of<AuthService>(context, listen: false);
       final apiService = ApiService(baseUrl: 'YOUR_API_BASE_URL', authService: authService);
       _data = apiService.fetchData('dashboard');
     }
   
     @override
     Widget build(BuildContext context) {
       return Scaffold(
         appBar: AppBar(
           title: Text("CropScope Dashboard"),
           actions: [
             IconButton(
               icon: Icon(Icons.logout),
               onPressed: () async {
                 await Provider.of<AuthService>(context, listen: false).logout();
                 Navigator.pushReplacement(context, MaterialPageRoute(builder: (context) => LoginPage()));
               },
             ),
           ],
         ),
         body: FutureBuilder<Map<String, dynamic>>(
           future: _data,
           builder: (context, snapshot) {
             if (snapshot.connectionState == ConnectionState.waiting) {
               return Center(child: CircularProgressIndicator());
             } else if (snapshot.hasError) {
               return Center(child: Text('Error: ${snapshot.error}'));
             } else if (!snapshot.hasData || snapshot.data!.isEmpty) {
               return Center(child: Text('No data available'));
             } else {
               final data = snapshot.data!;
               return Padding(
                 padding: const EdgeInsets.all(16.0),
                 child: Column(
                   children: <Widget>[
                     Text("${data['atmosphericTemperature']} °C Atmospheric Temperature", style: TextStyle(fontSize: 24)),
                     Text("${data['relativeHumidity']}% Relative Humidity", style: TextStyle(fontSize: 24)),
                     Text("${data['soilPH']} Soil pH", style: TextStyle(fontSize: 24)),
                     Text("${data['soilMoisture']}% Soil Moisture", style: TextStyle(fontSize: 24)),
                     SizedBox(height: 20),
                     ElevatedButton(
                       onPressed: () {
                         Navigator.push(context, MaterialPageRoute(builder: (context) => RecommendationsPage()));
                       },
                       child: Text("View Recommendations"),
                     ),
                   ],
                 ),
               );
             }
           },
         ),
       );
     }
   }
   ```

### Step 10: Implement Recommendations Page

1. **Create `recommendations.dart` in `lib/pages`:**
   
   ```dart
   import 'package:flutter/material.dart';
      import 'package:provider/provider.dart';
      import '../services/auth_service.dart';
      import '../services/api_service.dart';   
   class RecommendationsPage extends StatefulWidget {
        @override
        _RecommendationsPageState createState() => _RecommendationsPageState();
      } 
   class _RecommendationsPageState extends State<RecommendationsPage> {
        late Future<Map<String, dynamic>> _data;
   
   @override
    void initState() {
      super.initState();
      final authService = Provider.of<AuthService>(context, listen: false);
      final apiService = ApiService(baseUrl: 'YOUR_API_BASE_URL', authService: authService);
      _data = apiService.fetchData('recommendations');
    }
   
    @override
    Widget build(BuildContext context) {
      return Scaffold(
        appBar: AppBar(
          title: Text("Recommendations"),
        ),
        body: FutureBuilder<Map<String, dynamic>>(
          future: _data,
          builder: (context, snapshot) {
            if (snapshot.connectionState == ConnectionState.waiting) {
              return Center(child: CircularProgressIndicator());
            } else if (snapshot.hasError) {
              return Center(child: Text('Error: ${snapshot.error}'));
            } else if (!snapshot.hasData || snapshot.data!.isEmpty) {
              return Center(child: Text('No data available'));
            } else {
              final data = snapshot.data!;
              return Padding(
                padding: const EdgeInsets.all(16.0),
                child: Column(
                  children: <Widget>[
                    Text("Recommendations based on conditions:", style: TextStyle(fontSize: 24)),
                    ...data['recommendations'].map<Widget>((rec) {
                      return Text(rec, style: TextStyle(fontSize: 18));
                    }).toList(),
                  ],
                ),
              );
            }
          },
        ),
      );
    } 
    }
   ```

### Step 11: Main Application Entry

1. **Create `main.dart` in `lib`:**

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'services/auth_service.dart';
import 'pages/login_page.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MultiProvider(
      providers: [
        Provider<AuthService>(
          create: (_) => AuthService(baseUrl: 'YOUR_API_BASE_URL'),
        ),
      ],
      child: MaterialApp(
        title: 'CropScope',
        theme: ThemeData(
          primarySwatch: Colors.green,
        ),
        home: LoginPage(),
      ),
    );
  }
} 
```

### Step 12: Test the Application

1. **Run the App:**
   
   - Open terminal in your project directory and run:
     
     ```sh
     flutter run
     ```

2. **Verify Functionality:**
   
   - Enter login credentials and verify that the app navigates to the Dashboard on successful login.
   - Verify that the app shows an error page on failed login.
   - Check if the Dashboard and Recommendations pages display the fetched data correctly.

By following these steps, you should be able to set up, implement, and test the Flutter app as described. Adjust the API base URL and endpoint paths to match your backend configuration.
