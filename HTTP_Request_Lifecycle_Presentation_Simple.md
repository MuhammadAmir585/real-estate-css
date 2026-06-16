# HTTP Request Lifecycle
## PHP Travel Booking Platform - Simple Guide for Beginners

---

# 1. 🛬 Request Aata Hai (Request Comes In)

**Kya Hota Hai?**
- User apne browser se koi URL hit karta hai
- Example: `https://travel.com/login` ya `https://travel.com/flights`
- Request web server tak pohanchti hai

**Kaise Hota Hai?**
```
User Browser → Web Server (Apache/Nginx) → PHP Processor
```

**Asaan Shabd Mein:**
Jab aap booking.com par flights dhundo, to ek request browser se web server ko bhejdi jati hai.

---

# 2. 🔄 Web Server Request Receive Karta Hai

**Kya Hota Hai?**
- Web server (Apache ya Nginx) request receive karta hai
- Check karta hai ke ye kaunsa page chahiye
- PHP processor ko request pass karta hai

**Example:**
```
Browser: "Mujhe flights page chahiye"
↓
Web Server: "Ok, main PHP ko bulata hun"
↓
PHP: "Theek hai, main process karunga"
```

---

# 3. 🚀 Entry Point - index.php Khulta Hai

**Kya Hota Hai?**
- PHP `index.php` file ko khola jata hai
- Ye aapke application ka first file hota hai
- Yahan se sab kuch start hota hai

**index.php Kya Karta Hai?**
```
1. Output buffering on karta hai (response save karne ke liye)
2. config.php load karta hai
3. Database connection banana shuru karta hai
4. Global data load karta hai
```

**Zaroori Cheezein Yahan:**
- Session start
- Database variables
- Global settings

---

# 4. ⚙️ Configuration Load Hota Hai (config.php)

**Kya Hota Hai?**
- `.env` file se environment variables load hotay hain
- Database connection setup hoti hai
- Security headers set hotay hain
- i18n (language) system initialize hota hai

**config.php Mein Kya Hota Hai?**

### Step 1: Environment Variables
```
Database host, username, password
API keys
Security settings
```

### Step 2: Database Connection
```php
$db = new Medoo([
    'type' => 'mysql',
    'host' => 'localhost',
    'database' => 'travel_db',
    'username' => 'root',
    'password' => 'password'
]);
```

**Matlab:** Database se connection establish hota hai.

### Step 3: Security Headers Set Honay Hain
```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
```

**Matlab:** Attacks se bachne ke liye security headers set hotay hain.

### Step 4: Session Start
```php
if (session_status() === PHP_SESSION_NONE) {
    session_start();
}
```

**Matlab:** User ka session shuru hota hai (cookies, session variables).

### Step 5: Global Data Load
```php
$GLOBALS['app'] = settings
$GLOBALS['languages'] = available languages
$GLOBALS['currencies'] = currencies list
$GLOBALS['modules'] = active modules
```

**Matlab:** Aapke saare default data memory mein load hota hai.

---

# 5. 🌍 Global Data Initialization

**Kya Load Hota Hai?**

**A. Application Settings**
```
Home page title
Meta description
Status
```

**B. Languages**
```
English (default)
Arabic
French
Spanish
German
...aur bhi 6 languages
```

**C. Currencies**
```
USD
EUR
GBP
INR
AED
...aur bhi currencies
```

**D. Active Modules**
```
Flights booking
Hotels booking
Cars rental
Tours
Visa
Umrah
```

**Kyon Zaroori Hai?**
- Har request mein ye data use hota hai
- Isse database se baar-baar query nahi karni padti
- Application faster chalta hai

---

# 6. 🛣️ Router - URL Ko Match Karna

**Kya Hota Hai?**
- Request ka URL check hota hai
- Dekhte hain ke ye URL kaunse route se match karta hai
- Agar match hua to us route ka handler function chalaya jata hai

**Example:**

```
Request: GET /login
↓
Router Check: /login ka kaunsa handler hai?
↓
Found: loginRoutes.php mein GET /login handler hai
↓
Usko chalao!
```

**Aur Bhi Examples:**

| Request | Route | Handler |
|---------|-------|---------|
| GET /flights | `/flights` | Show flights page |
| POST /login | `/login` | Process login |
| GET /dashboard | `/dashboard` | Show dashboard |
| POST /api/flight/book | `/api/flight/book` | Book flight API |

**Routes Kahan Stored Hain?**
```
app/routes/
├── mainRoutes.php (homepage)
├── users/
│   ├── loginRoutes.php
│   ├── signupRoutes.php
│   └── profileRoutes.php
├── flights/
│   ├── bookingRoutes.php
│   └── listingRoutes.php
├── api/
│   └── flights/
│       └── searchRoutes.php
└── admin/
    └── dashboardRoutes.php
```

---

# 7. 🔐 Middleware - Security Checks

**Kya Hota Hai?**
- Request handler ke pehle security checks hoti hain
- Dekhta hai ke user authorized hai ya nahi
- Dekhta hai ke CSRF token valid hai ya nahi

**Security Checks Ke Steps:**

### Check 1: HTTP Method Validation
```
Allowed: GET, POST, PUT, DELETE
Not allowed: PATCH, OPTIONS, TRACE
↓
Invalid method? → 405 Method Not Allowed
```

### Check 2: CSRF Token Validation
```
POST request ayi?
↓
$_POST['csrf_token'] check karo
↓
Token session se match karta hai?
↓
Nahi match? → 403 Forbidden (request rejected)
```

**CSRF Kya Hai?**
- Cross-Site Request Forgery
- Hacker fake request bhejte hain aapke naam se
- Token se ye fake request rok di jati hai

### Check 3: Authentication Check
```
User logged in hai?
↓
$_SESSION['user_id'] exist karta hai?
↓
Nahi? → Login page par redirect karo
```

### Check 4: Authorization Check
```
User admin hai?
↓
$_SESSION['user_role'] == 'admin'?
↓
Nahi? → Access denied
```

**Real Example - Login Page:**
```
GET /login
↓
1. HTTP method valid? ✓ (GET allowed)
2. CSRF token check? ✓ (GET par nahi required)
3. Already logged in? (nahi to) ✓
4. Show login form ✓
```

**Real Example - Admin Dashboard:**
```
GET /admin/dashboard
↓
1. HTTP method valid? ✓ (GET allowed)
2. User logged in? ✓ (check $SESSION)
3. User is admin? ✓ (check role)
4. Show admin dashboard ✓
```

---

# 8. 📥 Input Extraction (Data Lena)

**Kya Hota Hai?**
- Form se ya request se data nikala jata hai
- GET request se: `$_GET`
- POST request se: `$_POST`
- JSON API se: `file_get_contents('php://input')`

**Example - Login Form:**

```html
<form method="POST" action="/login">
    <input type="email" name="email">
    <input type="password" name="password">
    <input type="hidden" name="csrf_token" value="...">
    <button>Login</button>
</form>
```

**PHP Mein:**
```php
$email = $_POST['email'];           // user@example.com
$password = $_POST['password'];     // user123
$csrf_token = $_POST['csrf_token']; // token123456...
```

**Example - Flight Booking API:**

```json
Request Body:
{
    "departure_city": "NEW",
    "arrival_city": "LHR",
    "departure_date": "2026-06-20",
    "passengers": 2
}
```

**PHP Mein:**
```php
$input = json_decode(file_get_contents('php://input'), true);
$from = $input['departure_city'];        // NEW
$to = $input['arrival_city'];            // LHR
$date = $input['departure_date'];        // 2026-06-20
$passengers = $input['passengers'];      // 2
```

---

# 9. ✅ Input Validation (Data Check Karna)

**Kya Hota Hai?**
- Extracted data ko check kiya jata hai
- Dekhte hain ke data valid format mein hai ya nahi
- Dekhte hain ke data safe hai ya nahi

**Validation Functions:**

### Email Validation
```php
if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
    throw Exception("Invalid email");
}
```

**Check:** 
- Format sahi hai? (name@domain.com)
- Email complete hai?

### Password Validation
```php
if (strlen($password) < 8) {
    throw Exception("Password 8 characters se kam nahi ho sakta");
}
```

**Check:**
- Password 8 characters se zyada hai?
- Empty to nahi hai?

### String Validation
```php
$name = trim(strip_tags($value));
if (mb_strlen($name) > 50) {
    throw Exception("Name 50 characters se zyada nahi ho sakta");
}
```

**Check:**
- Extra spaces remove karo
- HTML tags remove karo
- Length check karo
- Special characters check karo

### Integer Validation
```php
if (!is_numeric($number) || $number < 1 || $number > 100) {
    throw Exception("Invalid number");
}
```

**Check:**
- Sirf number hai?
- Min value se zyada hai?
- Max value se kam hai?

**Real Example - Login Validation:**

```php
// Step 1: Extract
$email = $_POST['email'] ?? '';
$password = $_POST['password'] ?? '';

// Step 2: Validate
if (empty($email)) {
    throw Exception("Email required");
}
if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
    throw Exception("Invalid email format");
}
if (empty($password)) {
    throw Exception("Password required");
}
if (strlen($password) < 8) {
    throw Exception("Password 8 characters se kam hai");
}

// Step 3: Agar sab valid hai to age badho
// ✓ All validation passed
```

---

# 10. 🔍 Database Query - User Ka Data Lena

**Kya Hota Hai?**
- Validated data use karke database query hoti hai
- Database se user ka record fetch kiya jata hai

**Login Example:**

```php
// Database query
$user = $db->get('users', '*', ['email' => $email]);

// Ye SQL generate hota hai:
// SELECT * FROM users WHERE email = 'user@example.com'
```

**Response Types:**

### User Mil Gaya:
```php
$user = [
    'id' => 123,
    'email' => 'user@example.com',
    'first_name' => 'John',
    'last_name' => 'Doe',
    'password' => 'hashed_password_123...',
    'status' => 'active',
    'email_verified' => true
]
```

### User Nahi Mila:
```php
$user = null

// Ya error
throw Exception("User not found");
```

**Query Kaise Secure Hai?**
```
Medoo ORM parameterized queries use karta hai

Bad (Vulnerable):
SELECT * FROM users WHERE email = '$email'

Good (Safe):
SELECT * FROM users WHERE email = ?
(Email separately pass hota hai, injection nahi ho sakta)
```

---

# 11. 🔐 Password Verification - Password Check Karna

**Kya Hota Hai?**
- Entered password check hota hai stored password se
- Passwords directly compare nahi hotay (hashed passwords)
- `password_verify()` function use hota hai

**Password Hashing Kya Hai?**

```
User: "mypassword123"
↓
Hashing Algorithm (bcrypt)
↓
Database: "$2y$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcg7b3XeKeUxWdeS86E36DvDDu2"
```

**Hashing Se Kya Faida?**
- Password readable nahi hota
- Database leak hone par bhi password safe rehta hai
- Hacker ko password nahi malum padta

**Verification Process:**

```php
$entered_password = $_POST['password'];        // "mypassword123"
$stored_hash = $user['password'];              // "$2y$10$N9q..."

if (password_verify($entered_password, $stored_hash)) {
    // Password sahi hai ✓
    // Login successful
} else {
    // Password galat hai ✗
    // Login failed
}
```

**Real Scenario:**

```
User: "mypassword123" enter karta hai
↓
password_verify() check karta hai
↓
"mypassword123" hash karo
↓
Naya hash == stored hash?
↓
Haan? → Login successful
Nahi? → Login failed
```

---

# 12. 🛡️ Account Security Checks

**Kya Check Hota Hai?**

### Check 1: Email Verified?
```php
if (!$user['email_verified']) {
    throw Exception("Please verify your email first");
}
```

### Check 2: Account Active?
```php
if ($user['status'] !== 'active') {
    throw Exception("Your account is not active");
}
```

### Check 3: Account Banned?
```php
if ($user['banned']) {
    throw Exception("Your account is banned");
}
```

### Check 4: Account Locked?
```php
if ($user['locked_until'] && strtotime($user['locked_until']) > time()) {
    throw Exception("Account locked due to too many failed attempts");
}
```

**Kyon Zaroori Hai?**
- Unauthorized login prevent karne ke liye
- Hacker se protect karne ke liye
- Account security ensure karne ke liye

---

# 13. ✅ Login Success - Session Set Karna

**Kya Hota Hai?**
- Login successful ho gya
- User ka session create hota hai
- User data memory mein store hota hai

**Session Variables Set Honay Hain:**

```php
$_SESSION['user_id'] = $user['id'];              // 123
$_SESSION['user_email'] = $user['email'];        // user@example.com
$_SESSION['user_name'] = $user['first_name'];    // John
$_SESSION['user_role'] = $user['role'];          // customer
$_SESSION['login_time'] = time();                // 1718618400
$_SESSION['user_logged_in'] = true;              // true flag
```

**Session Kya Hai?**
- User ki login state store karna
- Browser request ke saath cookies bhejne se session data server par rehta hai
- Har request pe session check hota hai
- Logout hone par session destroy hota hai

**"Remember Me" Cookie:**
```php
// 30 din ke liye auto-login
$cookieValue = base64_encode($userId . '|' . $hmac_signature);
setcookie('remember_me', $cookieValue, time() + (30 * 24 * 3600), '/', '', true, true);
```

**Matlab:** Jab user ek mahine baad aaye to woh automatically login ho jaye.

---

# 14. 📝 Activity Logging

**Kya Hota Hai?**
- Login ka record database mein save hota hai
- User ka IP address record hota hai
- Browser aur OS info record hota hai
- Country detect hota hai

**Log Record:**

```php
$db->insert('logs_users', [
    'user_id' => $user['user_id'],
    'type' => 'login',
    'description' => 'User logged in successfully',
    'user_ip' => '192.168.1.100',
    'user_agent' => 'Mozilla/5.0 (Windows...)',
    'created_at' => date('Y-m-d H:i:s')
]);
```

**Kyon Important Hai?**
- Security audit ke liye
- Suspicious activity detect karne ke liye
- User analytics ke liye
- Legal compliance ke liye

---

# 15. 🔗 Webhook Trigger Hota Hai

**Kya Hota Hai?**
- Login success event trigger hota hai
- Custom logic execute hota hai
- External services ko notification bhejdi jati hai

**Example Webhook:**

```php
triggerWebhook('users/login', 'login.success', [
    'user_id' => $user['user_id'],
    'email' => $user['email'],
    'first_name' => $user['first_name'],
    'timestamp' => date('Y-m-d H:i:s')
]);
```

**Webhook Kya Karta Hai?**
- Welcome email bhej sakta hai
- Notification send kar sakta hai
- Analytics update kar sakta hai
- External system ko notify kar sakta hai

**Webhook File Structure:**
```
app/webhooks/
├── users/
│   ├── login.php
│   ├── signup.php
│   └── logout.php
└── bookings/
    └── created.php
```

---

# 16. 📤 Response - Browser Ko Jawab Bhejta Hai

**Kya Hota Hai?**
- Login successful message generate hota hai
- Browser ko response bhejdi jati hai
- Browser ko next page dikhaya jata hai

**Response Types:**

### Type 1: Redirect Response
```php
header('Location: ' . root . 'dashboard');
exit;

// Matlab: Dashboard page par chalo
```

### Type 2: HTML Response
```php
require_once views."includes/header.php";
require_once views."flights.php";
require_once views."includes/footer.php";

// Matlab: Pura HTML page send karo
```

### Type 3: JSON Response (API)
```php
header('Content-Type: application/json');
echo json_encode([
    'success' => true,
    'user_id' => $user['id'],
    'message' => 'Login successful'
]);

// Matlab: JSON format mein data send karo
```

### Type 4: Error Response
```php
http_response_code(400);
echo json_encode([
    'success' => false,
    'message' => 'Invalid email or password'
]);

// Matlab: Error code aur message bhejo
```

---

# 17. 🌐 Browser Receives Response

**Kya Hota Hai?**
- Browser response receive karta hai
- Response ke status code check hota hai
- Response body render hota hai

**Status Codes:**

| Code | Matlab |
|------|--------|
| 200 | OK - Request successful ✓ |
| 301/302 | Redirect - Dusre page pe jao |
| 400 | Bad Request - Input galat ✗ |
| 401 | Unauthorized - Login required ✗ |
| 403 | Forbidden - Access denied ✗ |
| 404 | Not Found - Page nahi mila ✗ |
| 500 | Server Error - Kuch error ✗ |

**Browser Kya Render Karta Hai?**
- HTML → Web page show karta hai
- JSON → Application data display karta hai
- Redirect → Naya page load karta hai

---

# Complete Request Flow - Visual

```
1. 🛬 REQUEST AATI HAI
   User: Click on login button
   ↓

2. 🔄 WEB SERVER RECEIVE KARTA HAI
   Apache/Nginx: Request accepted
   ↓

3. 🚀 PHP PROCESSOR START
   PHP: index.php load
   ↓

4. ⚙️ CONFIG SETUP
   - Database connect
   - Security headers
   - Session start
   - Global data load
   ↓

5. 🛣️ ROUTER MATCH
   - URL check: /login
   - Handler found
   ↓

6. 🔐 MIDDLEWARE CHECKS
   - HTTP method: GET/POST? ✓
   - CSRF token valid? ✓
   - User authorized? ✓
   ↓

7. 📥 INPUT EXTRACTION
   - Email: $_POST['email']
   - Password: $_POST['password']
   ↓

8. ✅ VALIDATION
   - Email format valid?
   - Password length >= 8?
   - All checks passed?
   ↓

9. 🔍 DATABASE QUERY
   - Query: SELECT * FROM users WHERE email = ?
   - Result: User record found
   ↓

10. 🔐 PASSWORD VERIFY
    - password_verify(entered, stored)
    - Result: Password match ✓
    ↓

11. 🛡️ SECURITY CHECKS
    - Email verified? ✓
    - Account active? ✓
    - Not locked? ✓
    - Not banned? ✓
    ↓

12. ✅ LOGIN SUCCESS
    - Session set: $_SESSION['user_id'] = 123
    - Database update: last_login
    ↓

13. 📝 LOGGING
    - Activity log insert
    ↓

14. 🔗 WEBHOOK
    - Trigger: login.success
    ↓

15. 📤 RESPONSE
    - Redirect: /dashboard
    - HTTP Status: 302
    ↓

16. 🌐 BROWSER
    - Receives redirect
    - Loads dashboard page
    ↓

17. ✅ END - User logged in successfully!
```

---

# Flight Booking Example - Puri Process

```
1. 🛬 REQUEST AATI HAI
   User: POST /api/flight/booking/save-draft
   Body: { flight_data: {...} }
   ↓

2. ⚙️ CONFIG LOAD
   - Database ready
   ↓

3. 🛣️ ROUTER MATCH
   - Route found: /api/flight/booking/save-draft
   ↓

4. 🔐 MIDDLEWARE
   - CSRF token valid?
   - User authenticated?
   ↓

5. 📥 INPUT EXTRACT
   - JSON decode
   - flight_data = $input['flight_data']
   ↓

6. ✅ VALIDATION
   - flight_data exists?
   - Format valid?
   ↓

7. 🔍 DATABASE
   - Generate booking hash
   - INSERT logs_bookings
   ↓

8. 🔗 WEBHOOK
   - Trigger: flights.booking.draft_created
   ↓

9. 📤 RESPONSE
   JSON response:
   {
       "success": true,
       "hash": "abc123def456",
       "message": "Booking saved"
   }
   ↓

10. 🌐 BROWSER
    - Receives JSON
    - JavaScript processes data
    - Shows confirmation message
```

---

# Key Points (Yaad Rakho)

## 1. Request ka Safar
```
Browser → Web Server → PHP → Database → Response → Browser
```

## 2. Security Layers
```
- CSRF token check
- Authentication check
- Authorization check
- Input validation
- Password hashing
- Activity logging
```

## 3. Database Protect
```
- Parameterized queries (SQL injection prevent)
- Prepared statements
- Input sanitization
```

## 4. Session Management
```
- User login state store
- Session variables
- Remember me cookies
- Session timeout
```

## 5. Error Handling
```
- Try-catch blocks
- Exception throwing
- User-friendly messages
- Server logging
```

## 6. Logging & Monitoring
```
- Activity logs
- Error logs
- Performance metrics
- Webhook execution logs
```

---

# Common Questions

**Q: Request kahan se start hota hai?**
A: Browser se! Jab aap URL type karte ho ya button click karte ho, request bhejdi jati hai.

**Q: config.php kab run hota hai?**
A: Har request ke shuru mein! Pehla to config.php run hota hai, phir baaki code.

**Q: Session kya hota hai?**
A: Session mein user ki login info store hoti hai. Jab logout karte ho to session clear ho jata hai.

**Q: CSRF token kyo zaroori hai?**
A: Hackers fake requests bhej sakte hain aapke naam se. CSRF token se ye fake requests block ho jate hain.

**Q: Database query kaise secure hai?**
A: Medoo ORM parameterized queries use karta hai. Ye SQL injection attacks se protect karta hai.

**Q: Webhook kya kaam karta hai?**
A: Webhook events ko trigger karta hai. Jab login hota hai to login webhook execute hota hai.

**Q: Response types kaun kaun hain?**
A: HTML (web pages), JSON (APIs), Redirect (dusre page), Files (PDF/images).

---

# Summary

## Simple Steps Mein Request Lifecycle:

1. **User request bhejta hai** → Browser se URL click
2. **Web server receive karta hai** → Apache/Nginx process karta hai
3. **PHP start hota hai** → index.php load
4. **Config initialize** → Database, security, sessions
5. **Route match hota hai** → URL se sahi handler find
6. **Security checks** → CSRF, auth, authorization
7. **Input extract aur validate** → Data check karna
8. **Database query** → Data fetch/save karna
9. **Business logic** → Login verify, booking process
10. **Logging** → Activity record karna
11. **Webhook trigger** → Events notify karna
12. **Response generate** → HTML/JSON tayyar karna
13. **Browser receive** → Response show karna

---

**Aur zyada jaankari ke liye code dekho:**
- Entry: [index.php](index.php)
- Config: [config.php](config.php)
- Routes: [app/routes/](app/routes/)
- Auth: [app/lib/auth.php](app/lib/auth.php)
- Validation: [app/lib/validate_input.php](app/lib/validate_input.php)

---

**End of Presentation**
Shukrya dekhne ke liye! 🙏
