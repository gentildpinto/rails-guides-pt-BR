**NÃO LEIA ESTE ARQUIVO NO GITHUB, OS GUIAS SÃO PUBLICADOS NO https://guiarails.com.br.**
**DO NOT READ THIS FILE ON GITHUB, GUIDES ARE PUBLISHED ON https://guides.rubyonrails.org.**

Auto Carregamento e Recarregamento de Constantes (Modo Zeitwerk)
======================================================

Esse guia documenta como funciona o auto carregamento e o recarregamento no modo `zeitwerk`.

Depois de ler este guia, você saberá:

* Modos de Auto carregamento
* Configurações Relacionadas ao Rails
* Estrutura do projeto
* Auto carregamento, recarregamento e _eager loading_
* Herança de Tabela Única
* E mais

--------------------------------------------------------------------------------

Introdução
------------

INFO. Esse guia documenta auto carregamento no modo `zeitwerk`, que é algo novo no Rails 6. Se o que você quer ler é sobre o modo `classic`, leia [Auto Carregamento e Recarregamento de Constantes (Modo Classic)](autoloading_and_reloading_constants_classic_mode.html).

Em um programa Ruby normal, dependências precisam ser carregadas de forma manual. Por exemplo, o *controller* a seguir utiliza as classes `ApplicationController` e `Post`, e normalmente você precisaria colocar alguns `require` para utilizá-los:

```ruby
# NÃO FAÇA ISSO.
require "application_controller"
require "post"
# NÃO FAÇA ISSO.

class PostsController < ApplicationController
  def index
    @posts = Post.all
  end
end
```

Isso não é necessário em aplicações Rails, onde as classes e os módulos da aplicação estão disponíveis em todo canto.

```ruby
class PostsController < ApplicationController
  def index
    @posts = Post.all
  end
end
```

Comumente, aplicações Rails só utilizam `require` para carregar coisas da sua pasta `lib`, da biblioteca padrão do ruby, do Ruby gems, etc. Ou seja, qualquer coisa que não esteja presente nos caminhos de auto carregamento, explicado a seguir.


Habilitando o modo Zeitwerk
----------------------
O modo `zeitwerk` de auto carregamento é habilitado por padrão nas aplicações Rails 6 que utilizam CRuby:

```ruby
# config/application.rb
config.load_defaults 6.0 # habilita o modo zeitwerk usando CRuby
```

No modo `zeitwerk`, o Rails utiliza [Zeitwerk](https://github.com/fxn/zeitwerk) de forma interna para fazer carregamento automático, recarregamento e *eager load*. O Rails cria e configura uma instância Zeitwerk dedicada que gerencia o projeto.

INFO. Você não configura o Zeitwerk manualmente em uma aplicação Rails. Ao invés disso, você configura a aplicação usando uma configuração portável, que é descrita neste guia. Assim, o Rails traduz essas configurações para o Zeitwerk por você.

Estrutura do Projeto
-----------------

Em uma aplicação Rails, nomes de arquivos precisam corresponder às constantes que definem, com diretórios agindo como *namespaces*.

Por exemplo, o arquivo `app/helpers/users_helper.rb` deve definir `UsersHelper` e o arquivo `app/controllers/admin/payments_controller.rb` deve definir `Admin::PaymentsController`.

Por padrão o Rails configura o *Zeitwerk* para inflexionar nomes de arquivos com `String#camelize`. Por exemplo, é esperado que `app/controllers/users_controller.rb` defina a constante `UsersController` pois

```ruby
"users_controller".camelize # => UsersController
```

A seção _Customizando Inflexões_ abaixo documenta maneiras de sobrescrever esse comportamento padrão.

Por favor, verifique a [documentação do Zeitwerk](https://github.com/fxn/zeitwerk#file-structure) para mais detalhes.

Autoload Paths
--------------

Referimo-nos à lista de diretórios de aplicações cujos conteúdos devem ser carregados automaticamente como _caminhos de carregamento automático_. Por exemplo, `app/models`. Esses diretórios representam o namespace raiz: `Object`.

INFO. Os caminhos de carregamento automático são chamados de _diretórios raiz_ na documentação do Zeitwerk, mas continuaremos com "caminho de carregamento automático" neste guia.

Dentro de um caminho de carregamento automático, os nomes dos arquivos devem corresponder às constantes que eles definem conforme documentado [aqui](https://github.com/fxn/zeitwerk#file-structure).

Por padrão, os caminhos de carregamento automático de uma aplicação consistem em todos os subdiretórios de `app` que existem quando o aplicativo é inicializado --- exceto para `assets`, `javascript`,` views`, --- mais os caminhos de carregamento automático das *engines* que podem.

Por exemplo, se `UsersHelper` for implementado em `app/helpers/users_helper.rb`, o módulo é auto carregado, você não precisa (e não deve escrever) uma chamada `require` para ele:

```bash
$ bin/rails runner 'p UsersHelper'
UsersHelper
```

Os caminhos de carregamento automático escolhem automaticamente quaisquer diretórios personalizados em `app`. Por exemplo, se sua aplicação tem `app/presenters`, ou `app/services`, etc., eles são adicionados aos caminhos de carregamento automático.

O *array* de caminhos de carregamento automático pode ser estendido alterando `config.autoload_paths`, em `config/application.rb`, mas hoje em dia isso é desencorajado.

WARNING. Por favor, não altere `ActiveSupport::Dependencies.autoload_paths`, a interface pública para alterar os caminhos de carregamento automático é `config.autoload_paths`.


$LOAD_PATH
----------

Caminhos de carregamento automático são adicionados a `$LOAD_PATH` por padrão. No entanto, o Zeitwerk usa nomes de arquivo absolutos internamente, e sua aplicação não deve emitir chamadas `require` para arquivos carregáveis ​​automaticamente, então esses diretórios não são realmente necessários lá. Você pode cancelar com este sinalizador:

```ruby
config.add_autoload_paths_to_load_path = false
```

Isso pode acelerar um pouco as chamadas `require` legítimas, uma vez que há menos pesquisas. Além disso, se seu aplicativo usa [Bootsnap](https://github.com/Shopify/bootsnap), isso evita que a biblioteca crie índices desnecessários e economiza a RAM necessária.


Reloading
---------

Rails automatically reloads classes and modules if application files change.

More precisely, if the web server is running and application files have been modified, Rails unloads all autoloaded constants just before the next request is processed. That way, application classes or modules used during that request are going to be autoloaded, thus picking up their current implementation in the file system.

Reloading can be enabled or disabled. The setting that controls this behavior is `config.cache_classes`, which is false by default in `development` mode (reloading enabled), and true by default in `production` mode (reloading disabled).

Rails detects files have changed using an evented file monitor (default), or walking the autoload paths, depending on `config.file_watcher`.

In a Rails console there is no file watcher active regardless of the value of `config.cache_classes`. This is so because, normally, it would be confusing to have code reloaded in the middle of a console session, the same way you generally want an individual request to be served by a consistent, non-changing set of application classes and modules.

However, you can force a reload in the console by executing `reload!`:

```irb
irb(main):001:0> User.object_id
=> 70136277390120
irb(main):002:0> reload!
Reloading...
=> true
irb(main):003:0> User.object_id
=> 70136284426020
```

as you can see, the class object stored in the `User` constant is different after reloading.

### Reloading and Stale Objects

It is very important to understand that Ruby does not have a way to truly reload classes and modules in memory, and have that reflected everywhere they are already used. Technically, "unloading" the `User` class means removing the `User` constant via `Object.send(:remove_const, "User")`.

Therefore, code that references a reloadable class or module, but that is not executed again on reload, becomes stale. Let's see an example next.

Let's consider this initializer:

```ruby
# config/initializers/configure_payment_gateway.rb
# DO NOT DO THIS.
$PAYMENT_GATEWAY = Rails.env.production? ? RealGateway : MockedGateway
# DO NOT DO THIS.
```

The idea would be to use `$PAYMENT_GATEWAY` in the code, and let the initializer set that to the actual implementation depending on the environment.

On reload, `MockedGateway` is reloaded, but `$PAYMENT_GATEWAY` is not updated because initializers only run on boot. Therefore, it won't reflect the changes.

There are several ways to do this safely. For instance, the application could define a class method `PaymentGateway.impl` whose definition depends on the environment; or could define `PaymentGateway` to have a parent class or mixin that depends on the environment; or use the same global variable trick, but in a reloader callback, as explained below.

Let's see other situations that involve stale class or module objects.

Check this Rails console session:

```irb
irb> joe = User.new
irb> reload!
irb> alice = User.new
irb> joe.class == alice.class
=> false
```

`joe` is an instance of the original `User` class. When there is a reload, the `User` constant evaluates to a different, reloaded class. `alice` is an instance of the current one, but `joe` is not, his class is stale. You may define `joe` again, start an IRB subsession, or just launch a new console instead of calling `reload!`.

Another situation in which you may find this gotcha is subclassing reloadable classes in a place that is not reloaded:

```ruby
# lib/vip_user.rb
class VipUser < User
end
```

if `User` is reloaded, since `VipUser` is not, the superclass of `VipUser` is the original stale class object.

Bottom line: **do not cache reloadable classes or modules**.

### Autoloading when the application boots

Applications can safely autoload constants during boot using a reloader callback:

```ruby
Rails.application.reloader.to_prepare do
  $PAYMENT_GATEWAY = Rails.env.production? ? RealGateway : MockedGateway
end
```

That block runs when the application boots, and every time code is reloaded.

NOTE: For historical reasons, this callback may run twice. The code it executes must be idempotent.

However, if you do not need to reload the class, it is easier to define it in a directory which does not belong to the autoload paths. For instance, `lib` is an idiomatic choice, it does not belong to the autoload paths by default but it belongs to `$LOAD_PATH`. Then, in the place the class is needed at boot time, just perform a regular `require` to load it.

For example, there is no point in defining reloadable Rack middleware, because changes would not be reflected in the instance stored in the middleware stack anyway. If `lib/my_app/middleware/foo.rb` defines a middleware class, then in `config/application.rb` you write:

```ruby
require "my_app/middleware/foo"
...
config.middleware.use MyApp::Middleware::Foo
```

To have changes in that middleware reflected, you need to restart the server.


Eager Loading
-------------

In production-like environments it is generally better to load all the application code when the application boots. Eager loading puts everything in memory ready to serve requests right away, and it is also [CoW](https://en.wikipedia.org/wiki/Copy-on-write)-friendly.

Eager loading is controlled by the flag `config.eager_load`, which is enabled by default in `production` mode.

The order in which files are eager loaded is undefined.

if the `Zeitwerk` constant is defined, Rails invokes `Zeitwerk::Loader.eager_load_all` regardless of the application autoloading mode. That ensures dependencies managed by Zeitwerk are eager loaded.


Single Table Inheritance
------------------------

Single Table Inheritance is a feature that doesn't play well with lazy loading. Reason is, its API generally needs to be able to enumerate the STI hierarchy to work correctly, whereas lazy loading defers loading classes until they are referenced. You can't enumerate what you haven't referenced yet.

In a sense, applications need to eager load STI hierarchies regardless of the loading mode.

Of course, if the application eager loads on boot, that is already accomplished. When it does not, it is in practice enough to instantiate the existing types in the database, which in development or test modes is usually fine. One way to do that is to throw this module into the `lib` directory:

```ruby
module StiPreload
  unless Rails.application.config.eager_load
    extend ActiveSupport::Concern

    included do
      cattr_accessor :preloaded, instance_accessor: false
    end

    class_methods do
      def descendants
        preload_sti unless preloaded
        super
      end

      # Constantizes all types present in the database. There might be more on
      # disk, but that does not matter in practice as far as the STI API is
      # concerned.
      #
      # Assumes store_full_sti_class is true, the default.
      def preload_sti
        types_in_db = \
          base_class.
            unscoped.
            select(inheritance_column).
            distinct.
            pluck(inheritance_column).
            compact

        types_in_db.each do |type|
          logger.debug("Preloading STI type #{type}")
          type.constantize
        end

        self.preloaded = true
      end
    end
  end
end
```

and then include it in the STI root classes of your project:

```ruby
# app/models/shape.rb
require "sti_preload"

class Shape < ApplicationRecord
  include StiPreload # Only in the root class.
end
```

```ruby
# app/models/polygon.rb
class Polygon < Shape
end
```

```ruby
# app/models/triangle.rb
class Triangle < Polygon
end
```

Customizing Inflections
-----------------------

By default, Rails uses `String#camelize` to know which constant should a given file or directory name define. For example, `posts_controller.rb` should define `PostsController` because that is what `"posts_controller".camelize` returns.

It could be the case that some particular file or directory name does not get inflected as you want. For instance, `html_parser.rb` is expected to define `HtmlParser` by default. What if you prefer the class to be `HTMLParser`? There are a few ways to customize this.

The easiest way is to define acronyms in `config/initializers/inflections.rb`:

```ruby
ActiveSupport::Inflector.inflections(:en) do |inflect|
  inflect.acronym "HTML"
  inflect.acronym "SSL"
end
```

Doing so affects how Active Support inflects globally. That may be fine in some applications, but you can also customize how to camelize individual basenames independently from Active Support by passing a collection of overrides to the default inflectors:

```ruby
# config/initializers/zeitwerk.rb
Rails.autoloaders.each do |autoloader|
  autoloader.inflector.inflect(
    "html_parser" => "HTMLParser",
    "ssl_error"   => "SSLError"
  )
end
```

That technique still depends on `String#camelize`, though, because that is what the default inflectors use as fallback. If you instead prefer not to depend on Active Support inflections at all and have absolute control over inflections, configure the inflectors to be instances of `Zeitwerk::Inflector`:

```ruby
# config/initializers/zeitwerk.rb
Rails.autoloaders.each do |autoloader|
  autoloader.inflector = Zeitwerk::Inflector.new
  autoloader.inflector.inflect(
    "html_parser" => "HTMLParser",
    "ssl_error"   => "SSLError"
  )
end
```

There is no global configuration that can affect said instances, they are deterministic.

You can even define a custom inflector for full flexibility. Please, check the [Zeitwerk documentation](https://github.com/fxn/zeitwerk#custom-inflector) for further details.

Troubleshooting
---------------

The best way to follow what the loaders are doing is to inspect their activity.

The easiest way to do that is to throw

```ruby
Rails.autoloaders.log!
```

to `config/application.rb` after loading the framework defaults. That will print traces to standard output.

If you prefer logging to a file, configure this instead:

```ruby
Rails.autoloaders.logger = Logger.new("#{Rails.root}/log/autoloading.log")
```

The Rails logger is still not ready in `config/application.rb`, but it is in initializers:

```ruby
# config/initializers/log_autoloaders.rb
Rails.autoloaders.logger = Rails.logger
```

Rails.autoloaders
-----------------

The Zeitwerk instances managing your application are available at

```ruby
Rails.autoloaders.main
Rails.autoloaders.once
```

The former is the main one. The latter is there mostly for backwards compatibility reasons, in case the application has something in `config.autoload_once_paths` (this is discouraged nowadays).

You can check if `zeitwerk` mode is enabled with

```ruby
Rails.autoloaders.zeitwerk_enabled?
```

Differences with Classic Mode
-----------------------------

### Ruby Constant Lookup Compliance

`classic` mode cannot match constant lookup semantics due to fundamental limitations of the technique it is based on, whereas `zeitwerk` mode works like Ruby.

For example, in `classic` mode defining classes or modules in namespaces with qualified constants this way

```ruby
class Admin::UsersController < ApplicationController
end
```

was not recommended because the resolution of constants inside their body was brittle. You'd better write them in this style:

```ruby
module Admin
  class UsersController < ApplicationController
  end
end
```

In `zeitwerk` mode that does not matter anymore, you can pick either style.

The resolution of a constant could depend on load order, the definition of a class or module object could depend on load order, there was edge cases with singleton classes, oftentimes you had to use `require_dependency` as a workaround, .... The guide for `classic` mode documents [these issues](autoloading_and_reloading_constants_classic_mode.html#common-gotchas).

All these problems are solved in `zeitwerk` mode, it just works as expected, and `require_dependency` should not be used anymore, it is no longer needed.

### Less File Lookups

In `classic` mode, every single missing constant triggers a file lookup that walks the autoload paths.

In `zeitwerk` mode there is only one pass. That pass is done once, not per missing constant, and so it is generally more performant. Subdirectories are visited only if their namespace is used.

### Underscore vs Camelize

Inflections go the other way around.

In `classic` mode, given a missing constant Rails _underscores_ its name and performs a file lookup. On the other hand, `zeitwerk` mode checks first the file system, and _camelizes_ file names to know the constant those files are expected to define.

While in common names these operations match, if acronyms or custom inflection rules are configured, they may not. For example, by default `"HTMLParser".underscore` is `"html_parser"`, and `"html_parser".camelize` is `"HtmlParser"`.

### More Differences

There are some other subtle differences, please check [this section of _Upgrading Ruby on Rails_](upgrading_ruby_on_rails.html#autoloading) guide for details.

Classic Mode is Deprecated
--------------------------

By now, it is still possible to use `classic` mode. However, `classic` is deprecated and will be eventually removed.

New applications should use `zeitwerk` mode (which is the default), and applications being upgrade are strongly encouraged to migrate to `zeitwerk` mode. Please check the [_Upgrading Ruby on Rails_](upgrading_ruby_on_rails.html#autoloading) guide for details.

Opting Out
----------

Applications can load Rails 6 defaults and still use the classic autoloader this way:

```ruby
# config/application.rb
config.load_defaults 6.0
config.autoloader = :classic
```

That may be handy if upgrading to Rails 6 in different phases, but classic mode is discouraged for new applications.

`zeitwerk` mode is not available in versions of Rails previous to 6.0.
