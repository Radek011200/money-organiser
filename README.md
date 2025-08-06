# Money Organiser

A web application for managing personal finances, tracking expenses, and organizing financial data.

## Authentication Setup

### Regular Authentication (Email/Password)

The application supports standard email and password authentication. This functionality is now enabled and should work out of the box.

### Google Authentication

To enable Google authentication:

1. Go to the [Google Cloud Console](https://console.cloud.google.com/apis/credentials)
2. Create a new project or select an existing one
3. Navigate to "APIs & Services" > "Credentials"
4. Click "Create Credentials" > "OAuth client ID"
5. Select "Web application" as the application type
6. Add a name for your OAuth client
7. Add authorized JavaScript origins:
   - `http://localhost:8000` (for local development)
8. Add authorized redirect URIs:
   - `http://localhost:8000/auth/google/callback` (for local development)
9. Click "Create"
10. Copy the generated Client ID and Client Secret

Then update your backend `.env` file with these values:

```
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
GOOGLE_REDIRECT_URI=http://localhost:8000/auth/google/callback
```

## CORS Configuration

If you're running the frontend and backend on different domains, IPs, or ports, you may encounter CORS (Cross-Origin Resource Sharing) issues. The application is configured to allow requests from the following origins:

- `http://localhost:5173`
- `http://172.25.21.191:5173`
- `http://192.168.1.17:5173`

If you need to add additional origins, modify the `allowed_origins` array in `backend/config/cors.php`:

```php
'allowed_origins' => [
    'http://localhost:5173',
    'http://your-custom-domain:port',
],
```

## Running the Application

Make sure to run both the backend and frontend servers:

### Backend
```
cd backend
php artisan serve
```

### Frontend
```
cd frontend
npm run dev
```

### Running on Different IPs/Networks

If you need to access the application from other devices on your network:

#### Backend
```
cd backend
php artisan serve --host=0.0.0.0
```

#### Frontend
```
cd frontend
npm run dev -- --host
```

## Troubleshooting

### CSRF Token Issues

If you encounter issues with CSRF token retrieval, such as:

```
Access to XMLHttpRequest at 'http://your-api-url/sanctum/csrf-cookie' from origin 'http://your-frontend-url' has been blocked by CORS policy
```

or

```
GET http://your-api-url/sanctum/csrf-cookie net::ERR_FAILED 500 (Internal Server Error)
```

or

```
"message": "CSRF token mismatch."
```

If you specifically encounter this error when registering a new user, ensure that:

1. The 'register' route is not excluded from CSRF verification in both:
   - `app/Http/Middleware/VerifyCsrfToken.php` - remove 'register' from the $except array
   - `bootstrap/app.php` - ensure 'register' is not in the validateCsrfTokens except list

Make sure that:

1. Laravel Sanctum is properly installed and configured
2. The backend `.env` file has the following settings:
   ```
   APP_URL=http://your-backend-url:8000  # Must match your actual backend URL
   SESSION_DOMAIN=null
   SESSION_SAME_SITE=none
   SESSION_SECURE_COOKIE=false  # For local development without HTTPS
   ```
3. The frontend and backend domains are included in the `stateful` array in `config/sanctum.php`
4. The Sanctum guard is properly defined in `config/auth.php`:
   ```php
   'guards' => [
       'web' => [
           'driver' => 'session',
           'provider' => 'users',
       ],
       'sanctum' => [
           'driver' => 'sanctum',
           'provider' => 'users',
       ],
   ],
   ```
5. The CORS middleware is configured to run before the session middleware in `bootstrap/app.php`:
   ```php
   ->withMiddleware(function (Middleware $middleware) {
       // ...
       $middleware->priority([
           \Illuminate\Http\Middleware\HandleCors::class,
           \Illuminate\Session\Middleware\StartSession::class,
       ]);
   })
   ```

   This ensures that CORS headers are added before the session is started, which is crucial for cross-origin requests.

6. The frontend is properly configured to include the CSRF token in all requests. Create a custom Axios instance that automatically includes the CSRF token:
   ```javascript
   // src/services/axios.js
   import axios from 'axios';

   const axiosInstance = axios.create({
     baseURL: import.meta.env.VITE_API_URL,
     withCredentials: true,
     headers: {
       'Accept': 'application/json',
       'Content-Type': 'application/json'
     }
   });

   axiosInstance.interceptors.request.use(
     config => {
       // Get the XSRF-TOKEN cookie
       const xsrfToken = getCookie('XSRF-TOKEN');

       // If the cookie exists, add it to the headers
       if (xsrfToken) {
         config.headers['X-XSRF-TOKEN'] = decodeURIComponent(xsrfToken);
       }

       return config;
     },
     error => {
       return Promise.reject(error);
     }
   );

   // Helper function to get a cookie by name
   function getCookie(name) {
     const match = document.cookie.match(new RegExp('(^|;\\s*)(' + name + ')=([^;]*)'));
     return match ? match[3] : null;
   }

   export default axiosInstance;
   ```

   Then use this custom Axios instance in all your service files instead of creating separate instances.

### Ad Blockers and Privacy Extensions

Some ad blockers or privacy extensions may block requests to `/sanctum/csrf-cookie` which is required for CSRF protection. This can result in errors like:

```
GET http://your-api-url/sanctum/csrf-cookie net::ERR_BLOCKED_BY_CLIENT
```

If you encounter this issue, try:

1. Disabling ad blockers or privacy extensions for the application domain
2. Adding the domain to the whitelist in your ad blocker settings
3. Using a browser without ad blockers or privacy extensions for testing
