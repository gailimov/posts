# MVC

## Вступление

Речь пойдет об MVC применительно к веб-разработке, т.к. другого опыта программирования у меня пока нет.

Термины:

* Доменная-логика — примитивная логика, включающая в себя получение/установку значений (без сохранения), простую валидацию свойств (без обращения в хранилище).
* Бизнес-логика — любая другая логика приложения. Это может быть взаимодействие с хранилищем, отправка писем, различная валидация, проверка доступа и т.п.

## Model

При знакомстве с MVC-фреймворками, многие начинают думать, что модель — это про базу данных. Например в Ruby on Rails и его различных PHP-последователях (Yii, Laravel и т.д.) она представляет собой реализацию паттерна ActiveRecord. Т.е. таблица в базе данных проецируется на объект, предоставляя удобный способ взаимодействия. Такая модель умеет выбирать, изменять и валидировать себя. Работа с ней выглядит примерно так:

```
$post = new Post();
$post
    ->setTitle('About ActiveRecord')
    ->setContent('ActiveRecord is pretty awesome! It made me feel like an F1 driver!')
    ->save();
```

ActiveRecord является частью концепции RAD (Rapid Application Development). Но паттерн имеет свои недостатки:

1. Нарушение SOLID, а именно принципа единой ответственности. Как уже было сказано, модели ActiveRecord умеют и выбираться, и изменяться.
2. Бизнес-логика и взаимодействие с БД тесно связаны, что усложняет переход на другой тип хранилища.

На самом деле модель это обобщенное название, включающее в себя различные компоненты со своими обязанностями. Я использую модель, состояющую из частей, описанных ниже.

###  Domain Model (Entity)

Модель предметной области, она же доменная модель, она же сущность — это просто POPO (Plane Old PHP Object). Она не знает ничего о своем хранилище и вообще ей на всех пофиг :). Все что ее инетересует — она сама. Я предпочитаю анемичные модели (Anemic Domain Model), т.е. простой класс без какой-либо особой логики, состоящий из геттеров/сеттеров и максимум способный себя провалидировать. Пример сущности Post:

```
<?php

namespace blog\entities;

class Post extends Entity
{
    protected $id;
    protected $title;
    protected $content;
    protected $publicationDate;

    public function setId($id)
    {
        $this->id = $id;
        return $this;
    }

    public function getId()
    {
        return $this->id;
    }

    // геттеры/сеттеры для остальных свойств...
}
```

### Data Mapper

Data Mapper маппит (проецирует) таблицу БД на сущность, и отвечает за взаимодействие с хранилищем. Пример интерфейса маппера:

```
<?php

namespace blog\mappers;

use blog\entities\Entity;

interface Mapper
{
    public function getOne($criteria = []);

    public function get($criteria = [], $skip = null, $limit = null, $sort = []);

    public function save(Entity $entity);

    public function delete(Entity $entity);
}
```

Работа с моделью выглядит так:

```
$post = new Post();
$post
    ->setTitle('About Data Mapper')
    ->setContent('I was blind but now I see! Thank you, dear Data Mapper!');

$postMapper = new PostMapper();
$postMapper->save($post);
```

Теперь каждый объекс имеет свою зону ответственности. Бизнес-логика и взаимодействие с БД разделены и для перехода на другое хранилише, нам нужно только реализовать другой маппер.

### Service Layer

Сервис содержит в себе всю бизнес-логику и может служить прослойкой между контроллером и Data Mapper'ом. Я использую сервисный слой как способ разгрузки контроллеров. Обычно есть разная логика которую непонятно куда включить. В такие моменты меня выручают сервисы. Пример:

```
<?php

namespace blog\services;

class Post extends Service
{
    public function get($criteria = [], $page = 1, $perPage = 10, $sort = [])
    {
        $skip = ($page - 1) * $perPage;

        return $this->mapper->get($criteria, $skip, $perPage, $sort);
    }
}
```

## View

С представлением ситуация такая же как с моделями. Многие думают, что это просто темплейты. На самом деле View подготавливает данные для дальнейшего рендеринга и включает в себя логику отображения. Встречается также название ViewModel, я считаю их эквивалентными. Пример:

```
<?php

namespace blog\views;

use DateTime;

class Post extends View
{
    public function getUrl()
    {
        return $this->getRouter()->create('posts/view', ['id' => $this->getEntity()->getId()]);
    }

    public function getFormattedDate()
    {
        return (new DateTime($this->getEntity()->getPublicationDate()))->format('d m Y, H:i');
    }

    public function getIsoDate()
    {
        return (new DateTime($this->getEntity()->getPublicationDate()))->format(DateTime::ATOM);
    }
}
```

Ну а в самом теплейте будет что-то вроде такого:

```
<?php foreach ($posts as $post): ?>
    <h1><a href="<?= $post->getUrl() ?>"><?= $post->getTitle() ?></a></h1>

    <div class="content">
        <?= $post->getContent() ?>
    </div>

    <time datetime="<?= $post->getIsoDate() ?>"><?= $post->getFormattedDate() ?></time>
<?php endforeach ?>
```

## Controller

С контроллерами обычно все понятно. Но многие люди используют их как главный скрипт, т.е. для всего. Существует два подхода: тонкие контроллеры с тостыми моделями и толстые контроллеры (ТТУК — толстые, тупые, уродливые, контроллеры) с тонкими моделями. Т.к. моя модель состоит из нескольких частей, я предпочитаю держать контроллеры тонкими. Контроллеры могут взаимодействовать со всеми компонентами MV. Пример:

```
<?php

namespace blog\controllers;

use blog\services\Service;

class PostsController extends Controller
{
    public function indexAction()
    {
        $container = $this->getContainer();

        $page = (int) abs($this->getRequest()->getQuery('page', 1));

        $perPage = $container->get('config')['blog']['posts']['perPage'];

        $service = $container->get('app\services\Post');
        list($posts, $totalPosts) = $service->get([], $page, $perPage, ['publicationDate' => -1]);

        $posts = Service::prepare($posts, $container->get('blog\views\Post'));

        return $this->getResponse()->setContent(
            $this->render([
                'posts' => $posts,
                'totalPosts' => $totalPosts,
                'page' => $page,
                'perPage' => $perPage
            ])
        );
    }
}
```

## Рекомендации

Иногда в контроллерах встречаются вызовы Query Builder'a, например:

```
$posts = Post::model()->findAll(['isPublished' => true])
    ->skip($skip)
    ->limit($perPage)
    ->sort(['publicationDate' => -1]);
```

Вместо этого можно в модели создать отдельный метод:

```
class Post extends ActiveRecord
{
    ...

    public function getRecentPublished($skip, $limit)
    {
        return self::model()->findAll(['isPublished' => true])
            ->skip($skip)
            ->limit($limit)
            ->sort(['publicationDate' => -1]);
    }

    ...
}
```

И вызывать его в контроллере:

```
$posts = Post::model()->getRecentPublished($skip, $limit);
```

## Заключение

В PHP для реализации модели можно взять Doctrine. Она умеет автоматическую генерацию схемы БД по сущностям, предоставляет ORM, DBAL и MongoDB ODM. В роли Data Mapper'ов используются Repository (пока не знаю в чем разница). Входит в поставку Symfony.

В принципе MVC не предлагает однозначной реализации, так что конкретный способ разделения зависит от разработчика или его фреймворка. Смысл в том, чтобы бизнес-логика была отделена от представления и программистам разрабатывающим приложение было удобно.