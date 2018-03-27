---
layout: post
title:  "Рефакторинг проекта: выделяем Service Layer"
date:   2018-03-06 20:08:00 +0400
categories:
---

В [предыдущей части][new-project-old-problems] рассказывалось о проблемах, с
которыми довелось встретиться на новом проекте, раскрывались причины
рефакторинга. В качестве основной проблемы была названа сильная связанность
бизнес-логики со слоем представления, точнее присутствие кода бизнес-логики в
контроллерах. Решено было более чётко разделить эти два слоя. Между тем
выделяемая бизнес-логика в большей своей части является прикладной и будет
выноситься в [Service Layer]. В этой части рассмотрим, как выносился код
бизнес-логики из контроллеров.

### Для чего эта статья

Описывается опыт выделения бизнес-логики из контроллеров (слой представления) в
собственный слой бизнес-логики, а точнее в [Service Layer]. Приводится принятая
в проекте структура [Service Layer].

## Предметная область

Опишем очень упрощённо предметную область.

Приложение является веб-клиентом к платформе `ServicesPlatform` предоставления
некого контента `Content`. Пользователи `User` получают доступ к контенту через
веб-браузер. Чтобы получить полный доступ к контенту пользователи его покупают.
`Purchase` - покупка контента.

`Content` и `User` помимо прочего содержaт поле `uid` - идентификатор в
платформе.

`Purchase` содержит поле `content_uid` - идентификатор купленного контента в
платформе.

Контент получаем из платформы. Покупка контента также выполняется через вызовы к
платформе.

Ввиду определённых причин приходится хранить контент в локальной БД.
Периодическая синхронизация с платформой обеспечивает актуальность контента.

## Изначальный код

Приведём упрощённый пример - контроллер `ContentController`. Весь код для
простоты разместим в одном файле. В реальности вспомогательные методы могут
использоваться в нескольких контроллерах, и часто они выносятся либо в базовый
контроллер, либо в концерн.

{% highlight ruby %}
# app/controllers/content_controller.rb

class ContentController < ApplicationController
  before_action :authenticate_user!, only: [:purchase]

  # GET /content
  # List content for current user.
  def index
    @content = content_available_for_purchase
    @purchased_content = purchased_content
    @purchases = purchases
  end

  # POST /content/:id/purchase
  # Purchase content.
  def purchase
    content = content_available_for_purchase.find(params[:id])
    ServicesPlatformApi.purchase(current_user.uid, content.uid)

    redirect_to action: :index
  end

  private

  # Get uids of content purchased by current user.
  # @return [Array<Integer>]
  def purchased_content_uids
    @purchased_content_uids ||= purchases.pluck(:content_uid)
  end

  # Get current user purchases from our platform.
  # @return [Array<Purchase>]
  def purchases
    @purchases ||= if current_user.guest?
                     []
                   else
                     ServicesPlatformApi.purchases(current_user.uid)
                                        .map { |attrs| Purchase.new(attrs) }
                   end
  end

  def content_available_for_purchase
    # Users with demo status have access to restricted list of content.
    all_content = user.demo? ? Content.demo : Content.all
    all_content.where.not(uid: purchased_content_uids(user))
  end

  def purchased_content
    Content.where(uid: purchased_content_uids)
  end
end
{% endhighlight %}

В контроллере два экшена:
  * `index` - вывести контент для текущего пользователя: доступный для покупки и
    уже купленный с информацией о покупке. Для этого находим доступный для покупки
    контент, купленный контент и сами покупки.
  * `purchase` - выполнить покупку контента. В этом случае отправляем запрос
    покупки в платформу.

В контроллере есть проблема, о которой упоминал в [предыдущей части][new-project-old-problems].
Чтобы выполнить некоторые шаги экшенов в консоли, придётся предварительно
подготовить окружение. Например, чтобы в консоли получить купленный
пользователем контент, то есть вызывать `purchased_content`, в нашем упрощённом
случае нужно перенести в консоль методы `purchased_content`,
`purchased_content_uids`, `purchases` и `current_user`. В реальном проекте все
может быть намного запутаннее.

## Принятая структура Service Layer

Перед тем, как выделить [Service Layer], рассмотрим принятую в проекте структуру
слоя бизнес-логики.

Слой бизнес-логики в проекте представлен двумя подслоями: ядро бизнес-логики
(домен) и [Service Layer].

Для каждого подслоя отводится папка:
  * `app/models` - содержит сущности домена, ядро бизнес-логики. Организуется
    как "облегчённая" [Domain Model].
  * `app/services` - содержит прикладную логику, сценарии в виде [Service Layer].
    Это внешний подслой. Организуется как [Transaction Script], точнее
    _operation script_.

`app/models` в основном содержит привычные для Rails модели. В нашем случае
здесь находятся модели `Content`, `Purchase`, `User`.

`app/services` содержит прикладные сервисы, сценарии. Названия файлов имеют
суффикс `_service.rb`, например `users_service.rb`. В файлах содержаться модули
с процедурами - собственными методами (singleton methods) модуля. Каждая
процедура представляет сценарий.

Организация сценариев в виде обычных процедур замечательна тем, что поток данных
в них обозримый и контролируемый: данные приходят в основном только через
аргументы процедуры и через [слой доступа к данным][layered-architecture] (БД и
внешние сервисы). Если же взять к примеру метод контроллера, то этому методу
открывается без ограничений вся обширная область контроллера, многочисленные
методы и переменные, например, `current_user`. В этом отношении обычные
процедуры и функции проще классов.

Ещё одно свойство данных процедур - stateless. Это налагает определённые
ограничения, но вместе с этим делает их проще.

Модули группируют сценарии по определённому признаку. Например, `UsersService`
включает в себя сценарии по работе с пользователями.

{% highlight ruby %}
# app/services/users_service.rb

module UsersService
  class << self
    def register(user_data)
      # ...
    end

    def change_password(user, new_password)
      # ...
    end

    # ...
  end
end
{% endhighlight %}

`UsersService` - достаточно общее название, поэтому модуль со временем может
раздуться, о чём нам говорит [rubocop]. В этом случае часть сценариев можно
перенести в более специализированный модуль, например, методы выше можно вынести
в `UserAccountsService`. Но вначале, пока сценариев немного, разумно их
размещать в таком обобщённом модуле, выделяя более частные модули при
необходимости.

Для исключения дублирования модули прикладных сервисов могут содержать не только
методы конечных сценариев, но и промежуточных. Последние могут быть шагами,
которые выполняются в нескольких сценариях, в том числи и в других модулях.
Таким образом, подслой [Service Layer] в нашем случае также имеет слоистую
структуру.

## Выделяем Service Layer

Выделим прикладную логику из `ContentController` (был приведён выше).

В контроллере отметим следующие сценарии:
* получить доступный для покупки контент;
* получить купленный контент;
* получить покупки;
* купить контент.

В контроллере эти сценарии реализуются соответственно методами:
* `content_available_for_purchase`;
* `purchased_content`;
* `purchases`;
* `purchase`.

Выделим эти методы в `ContentService`.

{% highlight ruby %}
# app/services/content_service.rb

module ContentService
  ContentNotAvailableForPurchase = Class.new(StandardError)

  class << self
    # Get content available to user for purchase.
    # @param [User] user
    # @return [Content::ActiveRecord_Relation]
    def content_available_for_purchase(user)
      # Users with demo status have access to restricted list of content.
      all_content = user.demo? ? Content.demo : Content.all
      all_content.where.not(uid: purchased_content_uids(user))
    end

    # Get content purchased by user.
    # @param [User] user
    # @return [Content::ActiveRecord_Relation]
    def purchased_content(user)
      Content.where(uid: purchased_content_uids(user))
    end

    # Get user purchases.
    # @param [User] user
    # @return [Array<Purchase>]
    def purchases(user)
      return [] if user.guest?

      ServicesPlatformApi.purchases(user.uid)
                         .map { |attrs| Purchase.new(attrs) }
    end

    # Purchase content by user.
    # @param [User] user
    # @param [Content] content
    # @raise [UsersService::ContentNotAvailableForPurchase] if the content is
    #   not available for purchase.
    def purchase_content(user, content)
      check_content_available_for_purchase(user, content)
      ServicesPlatformApi.purchase(user.uid, content.uid)
    end

    private

    # Get uids of content purchased by user.
    # @param [User] user
    # @return [Array<Integer>]
    def purchased_content_uids(user)
      purchases(user).pluck(:content_uid)
    end

    def check_content_available_for_purchase(user, content)
      unless content_available_for_purchase(user).where(id: content.id).exists?
        raise ContentNotAvailableForPurchase
      end
    end
  end
end
{% endhighlight %}

Контроллер стал заметно проще, избавившись от деталей бизнес-логики:

{% highlight ruby %}
# app/controllers/content_controller.rb

class ContentController < ApplicationController
  before_action :authenticate_user!, only: [:purchase]

  # GET /content
  # List content for current user.
  def index
    @content = ContentService.content_available_for_purchase(current_user)
    @purchased_content = ContentService.purchased_content(current_user)
    @purchases = ContentService.purchases(current_user)
  end

  # POST /content/:id/purchase
  # Purchase content.
  def purchase
    content = Content.find(params[:id])

    begin
      ContentService.purchase_content(current_user, content)
    rescue ContentService::ContentNotAvailableForPurchase
      flash[:error] = 'The content is not available for purchase'
    end

    redirect_to action: :index
  end
end
{% endhighlight %}

Нам удалось выделить сценарии приложения в методы модуля. Эти методы-процедуры
самостоятельны, в отличие от методов внутри контроллера, их без труда можно
вызвать из консоли:

{% highlight ruby %}
[1] pry(main)> user = User.find(42)
[2] pry(main)> ContentService.purchased_content(user)
{% endhighlight %}

Но очевидно, куда важнее само явное разделение двух разных аспектов приложения,
представления и бизнес-логики, на два простых, однородных компонента. Это
значительно облегчает работу над проектом.

К сожалению, рефакторинг привнёс проблему с производительностью: при вызове
экшена `index` метод `ContentService.purchases` вызывается три раза. Этот метод
достаточно дорогой: он делает вызов во внешний сервис. В изначальном
`ContentController` во избежании этих затрат использовалась мемоизация,
благодаря чему метод вызывался один раз. В случае нашей реализации
[Service Layer] в виде процедур это сделать не получится - методы должны быть
stateless.

## Заключение

Бизнес-логику из контроллеров вынесли в [Service Layer]. Трудность выполнения
сценариев в Rails-консоли устранена. Однако, появилась проблема с
производительностью. В [следующей части][extract-app-context] будем её решать.

[new-project-old-problems]: {{site.baseurl}}{% post_url 2018-02-22-new-project-old-problems %}
[extract-app-context]: {{site.baseurl}}{% post_url 2018-03-19-extract-app-context %}
[Service Layer]: https://martinfowler.com/eaaCatalog/serviceLayer.html
[layered-architecture]: https://martinfowler.com/bliki/PresentationDomainDataLayering.html
[Domain Model]: https://martinfowler.com/eaaCatalog/domainModel.html
[Transaction Script]: https://martinfowler.com/eaaCatalog/transactionScript.html
[rubocop]: https://github.com/bbatsov/rubocop

[modeline]: # ( vim: set tw=80 spell spl=ru,en: )
