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