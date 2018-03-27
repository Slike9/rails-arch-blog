---
layout: post
title:  "Рефакторинг проекта: выделяем прикладной контекст"
date:   2018-03-19 20:08:00 +0400
categories:
---

[Раннее][extract-service-layer] выделили бизнес-логику из контроллеров, однако
столкнулись с проблемой производительности. В экшене вызывалось три сценария,
каждый из которых запрашивал у внешнего сервиса те же данные, что и другие. В
результате получали два лишних запроса во внешний сервис. Это чревато лишней
задержкой от нескольких десяток до нескольких сотен миллисекунд. Также это
создаёт дополнительную нагрузку на память из-за новых экземпляров одних и тех же
данных, что в конечном счёте опять сказывается на производительности. Похожая
ситуация также может сложиться, когда требуемые данные получаются при сложной
трансформации уже имеющихся данных. Можно заметить, проблема возникает на стыке
слоёв [доступа данных и бизнес-логики][layered-architecture].

### Для чего эта статья

Решается проблема избыточного обращения к внешним сервисам из слоя
бизнес-логики. Рассматриваются несколько вариантов решения.

## Варианты решения

### Объект сервиса

Проблема с методами модуля в качестве сценариев в том, что они должны быть
stateless, из-за чего мы не можем хранить данные между вызовами.

Можно сценарии разместить в объекте, где подобного ограничения нет.

{% highlight ruby %}
# app/services/content_service.rb

class ContentService
  def initialize(user)
    @user = user
  end

  # Get content available to user for purchase.
  # @return [Content::ActiveRecord_Relation]
  def content_available_for_purchase
    # Users with demo status have access to restricted list of content.
    all_content = @user.demo? ? Content.demo : Content.all
    all_content.where.not(uid: purchased_content_uids)
  end

  # Get content purchased by user.
  # @return [Content::ActiveRecord_Relation]
  def purchased_content
    Content.where(uid: purchased_content_uids)
  end

  # Get user purchases.
  # @return [Array<Purchase>]
  def purchases
    @purchases ||= if @user.guest?
                      []
                    else
                      ServicesPlatformApi.purchases(@user.uid)
                                         .map { |attrs| Purchase.new(attrs) }
                    end
  end

  private

  def purchased_content_uids
    @purchased_content_uids ||= purchases.pluck(:content_uid)
  end
end
{% endhighlight %}

{% highlight ruby %}
# app/controllers/content_controller.rb

class ContentController < ApplicationController
  # GET /content
  # List content for current user.
  def index
    content_service = ContentService.new(current_user)
    @content = content_service.content_available_for_purchase
    @purchased_content = content_service.purchased_content
    @purchases = content_service.purchases
  end
end
{% endhighlight %}

Однако выбор методов модуля для сценариев был сознательным, о чём упоминалось
[раннее][extract-service-layer]. Далее рассматриваются только варианты на
методах модуля.

### Один сценарий на весь экшен

Методы сценариев внутри модуля не связаны, у них нет общей области с данными,
как это могло бы быть в случае с объектом сервисов. От чего методам приходится
самостоятельно получать нужные данные. Решением может быть объединение
нескольких сценариев, которые должны быть вызваны совместно, в один.

{% highlight ruby %}
# app/services/content_service.rb
module ContentService
  class << self
    def user_content_and_purchases(user)
      purchases = purchases(user)
      purchased_content_uids = purchases.pluck(:content_uid)

      all_content = user.demo? ? Content.demo : Content.all
      content_available_for_purchase = all_content.where.not(uid: purchased_content_uids)

      purchased_content = Content.where(uid: purchased_content_uids)

      {
        content_available_for_purchase: content_available_for_purchase,
        purchased_content: purchased_content,
        purchases: purchases
      }
    end

    # Get user purchases.
    # @param [User] user
    # @return [Array<Purchase>]
    def purchases(user)
      return [] if user.guest?

      ServicesPlatformApi.purchases(user.uid)
                         .map { |attrs| Purchase.new(attrs) }
    end
  end
end
{% endhighlight %}

{% highlight ruby %}
# app/controllers/content_controller.rb

class ContentController < ApplicationController
  # GET /content
  # List content for current user.
  def index
    result = ContentService.user_content_and_purchases(user)
    @content = result[:content]
    @purchased_content = result[:purchased_content]
    @purchases = result[:purchases]
  end
end
{% endhighlight %}

Основной недостаток данного подхода - бизнес-логика зависит от представления.
Если набор данных, который нужен экшену, изменится, то придётся менять и
сценарий.

Получение разных данных в методе сценария, по сути выполнение разных действий,
также не добавляет привлекательности. Такой код склонен к распуханию.

### Передаём нужные данные через аргументы

Решено не использовать один сценарий на весь экшен, имеющийся набор сценариев
оставляем.

Есть вариант передавать "проблемные" данные через аргументы.

{% highlight ruby %}
# app/services/content_service.rb

module ContentService
  class << self
    # Get content available to user for purchase.
    # @param [User] user
    # @param [Array<Integer>] purchased_content_uids
    # @return [Content::ActiveRecord_Relation]
    def content_available_for_purchase(user, purchased_content_uids)
      # Users with demo status have access to restricted list of content.
      all_content = user.demo? ? Content.demo : Content.all
      all_content.where.not(uid: purchased_content_uids)
    end

    # Get content purchased by user.
    # @param [Array<Integer>] purchased_content_uids
    # @return [Content::ActiveRecord_Relation]
    def purchased_content(purchased_content_uids)
      Content.where(uid: purchased_content_uids)
    end

    # Get user purchases.
    # @param [User] user
    # @return [Array<Purchase>]
    def purchases(user)
      return [] if user.guest?

      ServicesPlatformApi.purchases(user.uid)
                         .map { |attrs| Purchase.new(attrs) }
    end
  end
end
{% endhighlight %}

И в экшене перед вызовом сценариев следует получить нужные данные.

{% highlight ruby %}
# app/controllers/content_controller.rb

class ContentController < ApplicationController
  # GET /content
  # List content for current user.
  def index
    purchases = ContentService.purchases(current_user)
    purchased_content_uids = purchases.pluck(:content_uid)

    @content = ContentService.content_available_for_purchase(current_user, purchased_content_uids)
    @purchased_content = ContentService.purchased_content(purchased_content_uids)
    @purchases = purchases
  end
end
{% endhighlight %}

Главная проблема - API сценариев стал менее удобен, детали реализации сценариев
вылезли наружу. Клиенту для вызова сценариев приходится предварительно получать и
затем передавать в сценарии странные для него данные.

### Прикладной контекст

Пришли к тому, что следует сохранить набор сценариев, продиктованный предметной
областью, а не представлением, и при этом не ухудшить API, не заставлять клиента
знать и делать лишнее.

Необходимые данные в сценарии всё-таки как-то передавать надо. Значит надо
как-то скрывать лишние подробности от клиента.

Скрыть особенности получения и хранения данных может специальный объект -
_прикладной контекст_. Клиент создаёт прикладной контекст и передаёт его
сценариям. Сценарии уже у контекста запрашивают нужные данные.

Похожими вещами занимаются _репозитории_, но цели отличаются. Репозитории
предоставляют "законный", системный способ организации доступа к данным.
Прикладной контекст - это средство оптимизации, он должен быть максимально
тонким, не содержать большой логики, хранить только часто используемые данные.
Репозитории, впрочем, тоже способны помочь справиться с проблемой.

С точки зрения организации, вернее [слоистой архитектуры][layered-architecture],
прикладной контекст является частью слоя бизнес-логики, скорее даже
[Service Layer]. У нас файл контекста хранится в папке `app/services`.

## Введение прикладного контекста

В нашем случае мы вводим специализированный контекст - `UserContext`. Во-первых,
у нас часто используемые данные связаны с пользователем. Во-вторых, это даёт
возможность использовать более короткие имена методов в контексте: `purchases`
вместо `user_purchases` и т. д. В нашем рабочем проекте также было достаточно
`UserContext`.

{% highlight ruby %}
# app/services/user_context.rb

class UserContext
  attr_reader :user

  def initialize(user)
    @user = user
  end

  def purchases
    @purchases ||= if @user.guest?
                     []
                   else
                     ServicesPlatformApi.purchases(@user.uid)
                                        .map { |attrs| Purchase.new(attrs) }
                   end
  end

  def purchased_content_uids
    @purchased_content_uids ||= purchases.pluck(:content_uid)
  end
end
{% endhighlight %}

Методы в контексте получают и кэшируют данные, используя мемоизацию.

В контексте совсем без логики не обошлось, например в методе `purchases`.

Теперь в сценарии вместо пользователя `user` передаём контекст `user_context`.

{% highlight ruby %}
# app/services/content_service.rb

module ContentService
  class << self
    # Get content available to user for purchase.
    # @param [UserContext] user_context
    # @return [Content::ActiveRecord_Relation]
    def content_available_for_purchase(user_context)
      # Users with demo status have access to restricted list of content.
      user = user_context.user
      all_content = user.demo? ? Content.demo : Content.all
      all_content.where.not(uid: user_context.purchased_content_uids)
    end

    # Get content purchased by user.
    # @param [UserContext] user_context
    # @return [Content::ActiveRecord_Relation]
    def purchased_content(user_context)
      Content.where(uid: user_context.purchased_content_uids)
    end

    # Get user purchases.
    # @param [UserContext] user_context
    # @return [Array<Purchase>]
    def purchases(user_context)
      user_context.purchases
    end
  end
end
{% endhighlight %}

Честно говоря, API немного пострадало. Появляется необходимость иметь дело с
новой сущностью - контекстом. Также, например, вызов `purchases(user)` куда
проще, понятнее, чем `purchases(user_context)`.

API также стало неустойчивым: при потребности в каких-то новых данных может
понадобиться поменять параметр `user` на `user_context` или наоборот. Для
устранения неустойчивости можно везде вместо `user` использовать `user_context`.

Приходится идти на компромиссы. Главное, детали доступа к данным внутри
сценариев всё же скрыты от клиента.

{% highlight ruby %}
# app/controllers/content_controller.rb

class ContentController < ApplicationController
  # GET /content
  # List content for current user.
  def index
    @content = ContentService.content_available_for_purchase(user_context)
    @purchased_content = ContentService.purchased_content(user_context)
    @purchases = ContentService.purchases(user_context)
  end

  private

  def user_context
    @user_context ||= UserContext.new(current_user)
  end
end
{% endhighlight %}

Метод контроллера `user_context` обычно используется во многих местах, поэтому
он выносится в базовый контроллер.

Вызов сценария в консоли теперь выглядит так:

{% highlight ruby %}
[1] pry(main)> user = User.find(42)
[2] pry(main)> user_context = UserContext.new(user)
[3] pry(main)> ContentService.purchased_content(user_context)
{% endhighlight %}

Поток данных остался простым и наглядным.

## Заключение

Проблема с лишними обращениями в сторонний сервис решена использованием
прикладного контекста. Однако API сценариев немного пострадало.

В этой и [предыдущей][extract-service-layer] части неоднократно упоминался
[слой доступа к данным][layered-architecture]. Далее обратимся к нему.

[extract-service-layer]: {{site.baseurl}}{% post_url 2018-03-06-extract-service-layer %}
[layered-architecture]: https://martinfowler.com/bliki/PresentationDomainDataLayering.html
[Service Layer]: https://martinfowler.com/eaaCatalog/serviceLayer.html

[modeline]: # ( vim: set tw=80 spell spl=ru,en: )
