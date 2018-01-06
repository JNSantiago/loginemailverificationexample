### Adicionar coluna de verificação na migração de criar tabela de usuarios

$table->boolean('verified')->default(false);

### Adicionar tabela verification_tokens

php artisan make:migration create_verification_tokens_table

$table->integer('user_id')->unsigned()->index();
$table->string('token');
$table->foreign('user_id')->references('id')->on('users')->onDelete('cascade');

php artisan migrate