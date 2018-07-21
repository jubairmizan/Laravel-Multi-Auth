# Laravel-Multi-Auth
Laravel Multiple Authentication


Step 1 : set your route: 

Auth::routes();
/*Admin*/
Route::prefix('admin')->group(function() {
	Route::get('/login', 'Auth\AdminLoginController@showLoginForm')->name('admin.login');
	Route::post('/login', 'Auth\AdminLoginController@login')->name('admin.login.submit');
	Route::get('/', 'AdminController@index')->name('admin.dashboard');
});


Step 2 : Set Auth guards and providers "Config/auth.php"
  'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],

        'api' => [
            'driver' => 'token',
            'provider' => 'users',
        ],
        'admin' => [
            'driver' => 'session',
            'provider' => 'admins',
        ],

        'admin-api' => [
            'driver' => 'token',
            'provider' => 'admins',
        ],
    ],
    
    'providers' => [
        'users' => [
            'driver' => 'eloquent',
            'model' => App\User::class,
        ],
        'admins' => [
            'driver' => 'eloquent',
            'model' => App\Admin::class,
        ],
    ],
    
    
    'passwords' => [
        'users' => [
            'provider' => 'users',
            'table' => 'password_resets',
            'expire' => 60,
        ],
        'admins' => [
            'provider' => 'admins',
            'table' => 'password_resets',
            'expire' => 60,
        ],
    ],
Step 3 : Migrate Admins Table 
    Schema::create('admins', function (Blueprint $table) {
            $table->increments('id');
            $table->string('name');
            $table->string('email')->unique();
            $table->string('password');
            $table->string('position');
            $table->rememberToken();
            $table->timestamps();
        });
        
Step 4 : Controller App\Http\Controller\auth AdminLoginController.php
      <?php

      namespace App\Http\Controllers\Auth;

      use Illuminate\Http\Request;
      use App\Http\Controllers\Controller;
      use Auth;

      class AdminLoginController extends Controller
      {
          public function __construct()
          {
            $this->middleware('guest:admin');
          }

          public function showLoginForm()
          {
            return view('auth.admin-login');
          }

          public function login(Request $request)
          {
            // Validate the form data
            $this->validate($request, [
              'email'   => 'required|email',
              'password' => 'required|min:6'
            ]);

            // Attempt to log the user in
            if (Auth::guard('admin')->attempt(['email' => $request->email, 'password' => $request->password], $request->remember)) {
              // if successful, then redirect to their intended location
              return redirect()->intended(route('admin.dashboard'));
            }

            // if unsuccessful, then redirect back to the login with the form data
            return redirect()->back()->withInput($request->only('email', 'remember'));
          }
      }


Step 5 : Admin Controller App\Http\Controllers AdminController.php
      <?php

      namespace App\Http\Controllers;

      use Illuminate\Http\Request;

      class AdminController extends Controller
      {
          /**
           * Create a new controller instance.
           *
           * @return void
           */
          public function __construct()
          {
              $this->middleware('auth:admin');
          }

          /**
           * Show the application dashboard.
           *
           * @return \Illuminate\Http\Response
           */
          public function index()
          {
              return view('admin-dashboard');
          }
          public function adminLogout(){
              Auth::guard('admin')->logout();
              return redirect('/');
          }
      }
      
      
 Step 6 : Model Set Admin.php
     <?php

      namespace App;

      use Illuminate\Notifications\Notifiable;
      use Illuminate\Foundation\Auth\User as Authenticatable;

      class Admin extends Authenticatable
      {
          use Notifiable;

          /**
           * The attributes that are mass assignable.
           *
           * @var array
           */
          protected $fillable = [
              'name', 'email', 'password','position'
          ];

          /**
           * The attributes that should be hidden for arrays.
           *
           * @var array
           */
          protected $hidden = [
              'password', 'remember_token',
          ];
      }

Step 7 : Set Exception App\Exception Handler.php

    /* At the Top  */
    use Illuminate\Auth\AuthenticationException;

    /* At The Bottom Add Those Code */
    protected function unauthenticated($request, AuthenticationException $exception)
    {
        if ($request->expectsJson()) {
            return response()->json(['error' => 'Unauthenticated.'], 401);
        }

        $guard = array_get($exception->guards(), 0);

        switch ($guard) {
          case 'admin':
            $login = 'admin.login';
            break;

          default:
            $login = 'login';
            break;
        }
        return redirect()->guest(route($login));
    }
    
 Step 8 : Middleware App\Http\Middleware  RedirectIfAuthenticated.php
 
    switch ($guard) {
        case 'admin':
          if (Auth::guard($guard)->check()) {
            return redirect()->route('admin.dashboard');
          }
          break;

        default:
          if (Auth::guard($guard)->check()) {
              return redirect('/home');
          }
          break;
      }

      return $next($request);
