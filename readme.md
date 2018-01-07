### Gerar a autenticação do Laravel
```php
php artisan make:auth

```

### Adicionar coluna de verificação na migração de criar tabela de usuarios

```php
$table->boolean('verified')->default(false);
```

### Adicionar tabela verification_tokens

```php
php artisan make:migration create_verification_tokens_table

$table->integer('user_id')->unsigned()->index();
$table->string('token');
$table->foreign('user_id')->references('id')->on('users')->onDelete('cascade');


php artisan migrate
```

### Adicionar relacionamentos

```php
//User model
public function verificationToken()
{
    return $this->hasOne(VerificationToken::class);
}

public function hasVerifiedEmail()
{
    return $this->verified;
}

public static function byEmail($email)
{
    return static::where('email', $email);
}
```

```php
//VerificationToken model

public function user()
{
	return $this->belongsTo(User::class);
}

public function getRouteKeyName()
{
	return 'token';
}
```

### Configurando o controller de verificação e as rotas

```php
php artisan make:controller VerificationController

class VerificationController extends Controller
{
    public function verify(VerificationToken $token)
    {
    	//
    }

    public function resend(Request $request)
    {
    	//
    }
}
```

```php
Route::get('/verify/token/{token}', 'Auth\VerificationController@verify')->name('auth.verify'); 
Route::get('/verify/resend', 'Auth\VerificationController@resend')->name('auth.verify.resend');
```

### Alterar o auth controller

```php
// RegisterController
// Deslogar o usuario após o cadastro
protected function registered(Request $request, $user)
{
    $this->guard()->logout();

    return redirect('/login')->withInfo('Please verify your email');
}
```

```php
// Garantir que o usuario nao ira se logar caso nao tenha realizado a verificação de login
protected function authenticated(Request $request, $user)
{
    if(!$user->hasVerifiedEmail()) {
        $this->guard()->logout();

        return redirect('/login')
            ->withError('Please activate your account. <a href="' . route('auth.verify.resend') . '?email=' . $user->email .'">Resend?</a>');
    }
}
```

```php
// Deslogar o usuário caso o mesmo solicite trocar senha e ainda nao houver verificado o email
protected function sendResetResponse($response)
{
    if(!$this->guard()->user()->hasVerifiedEmail()) {
        $this->guard()->logout();
        return redirect('/login')->withInfo('Password changed successfully. Please verify your email');
    }
    return redirect($this->redirectPath())
                        ->with('status', trans($response));
}
```

### Registrar um Service Provider para Eloquent Events

```php
php artisan make:provider EloquentEventServiceProvider
```

```php
public function boot()
{
    User::created(function($user) {

        $token = $user->verificationToken()->create([
            'token' => bin2hex(random_bytes(32))
        ]);

        event(new UserRegistered($user));
    });
}
```

### Criar um evento UserRegistered

```php
//gerar um event basico
php artisan event:generate
```

```php
// Criar na pasta Events
use App\User;

class UserRegistered
{
    use SerializesModels;
    public $user;

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct(User $user)
    {
        $this->user = $user;
    }
}
```

### Criar um evento UserRequestedVerificationEmail

```php
class UserRequestedVerificationEmail
{
    use SerializesModels;
    public $user;

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct(User $user)
    {
        $this->user = $user;
    }
}
```

### Criar um listener

```php
//Na pasta Listeners

class SendVerificationEmail
{
    /**
     * Create the event listener.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Handle the event.
     *
     * @param  Event  $event
     * @return void
     */
    public function handle($event)
    {
        Mail::to($event->user)->send(new SendVerificationToken($event->user->verificationToken));
    }
}
```

```php
// Inserir no array: $listen do Proveiders/EventServiceProvider 
protected $listen = [
    'App\Events\UserRegistered' => [
        'App\Listeners\SendVerificationEmail',
    ],
    'App\Events\UserRequestedVerificationEmail' => [
        'App\Listeners\SendVerificationEmail',
    ],
];

```

### Criar uma classe de Email

```php
php artisan make:mail SendVerificationToken
```

```php
public $token;

public function __construct(VerificationToken $token)
{
    $this->token = $token;
}

public function build()
{
    return $this->subject('Please verify your email')
            ->view('email.auth.verification');
}
```

```html
To verify your account, visit the following link. <br> <br>

<a href="{{ route('auth.verify', $token) }}">Verify now</a>
```

### Verificando o usuario
```php
// no arquivo Controllers/auth/VerificationController.php
public function verify(VerificationToken $token)
    {
        $token->user()->update([
            'verified' => true
        ]);

        $token->delete();

        // Uncomment the following lines if you want to login the user 
        // directly upon email verification
        // Auth::login($token->user);
        // return redirect('/home');

        return redirect('/login')->withInfo('Email verification succesful. Please login again');
    }
```

### Reenviar verificação de email

```php
// no arquivo Controllers/auth/VerificationController.php
public function resend(Request $request)
    {
        $user = User::byEmail($request->email)->firstOrFail();

        if($user->hasVerifiedEmail()) {
            return redirect('/home');
        }

        event(new UserRequestedVerificationEmail($user));

        return redirect('/login')->withInfo('Verification email resent. Please check your inbox');
    }
```