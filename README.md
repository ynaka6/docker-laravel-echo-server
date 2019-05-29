# docker-laravel-echo-server

Laravel Echo ServerのDocker開発環境

## Description
Dockerを利用した Laravel Echo Server の開発環境
- PHP7.3
- MySQL8.0
- nginx
- composer
- redis
- node
- laravel-echo-server
- php-worker


## Usage
### Git clone
```
$ git clone xxxxx
$ cd docker-laravel-echo-server
```

### Docker compose build
```
$ docker-compose up -d
```


### Install Laravel 5 using Composer
```
$ docker-compose run composer create-project --prefer-dist "laravel/laravel=5.8.*" .
$ docker-compose run composer require predis/predis
```

### Setup Laravel
```
$ docker-compose exec app sh
$ sed -i -e "s/DB_HOST=.*/DB_HOST=db/" .env
$ sed -i -e "s/REDIS_HOST=.*/REDIS_HOST=redis/" .env
$ sed -i -e "s/BROADCAST_DRIVER=.*/BROADCAST_DRIVER=redis/" .env
```

### Migration
```
$ docker-compose exec app sh
$ php artisan migrate
```

### Setup Frontend
#### NPM
```
$ docker-compose run node npm install
$ docker-compose run node npm install --save-dev laravel-echo socket.io-client
$ docker-compose run node npm run dev
```

#### yarn
```
$ docker-compose run node yarn install
$ docker-compose run node yarn add --dev laravel-echo socket.io-client
$ docker-compose run node yarn run dev
```

## Getting Started
### Laravel側の設定
#### config/app.phpのprovidersの配列内にあるBroadcastServiceProviderのコメントアウトを外す
```
App\Providers\BroadcastServiceProvider::class,
```

#### PublicなEventを追加
```
$ docker-compose run app php artisan make:event PublicEvent
```

app/Events/PlublicEvent.phpが作成されます。  
内容は以下のようにします。

```
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Queue\SerializesModels;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;

class PublicEvent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Get the channels the event should broadcast on.
     *
     * @return \Illuminate\Broadcasting\Channel|array
     */
    public function broadcastOn()
    {
        return new Channel('public-event');
    }

    public function broadcastWith()
    {
        return [
            'message' => 'PUBLIC',
        ];
    }
}
```

#### EventのトリガーとなるURLを用意
routes/web.phpに以下のルーティングを追加します。

```
Route::get('/public-event', function(){
    broadcast(new \App\Events\PublicEvent);
    return 'public';
});
```

### フロントエンド（Javascrip）の設定
#### resources/js/app.jsの修正
laravel-echoからのイベントを受け付けるため、app.jsを以下のように修正します。

```
import Echo from "laravel-echo";

window.io = require("socket.io-client");
window.Echo = new Echo({
    broadcaster: "socket.io",
    host: window.location.hostname + ":6001"
});

window.Echo.channel("laravel_database_public-event").listen(
    "PublicEvent",
    e => {
        console.log(e);
    }
);
```

#### jsをビルド
```
$ docker-compose run node npm run dev
```


#### welcomeページでapp.jsを読み込む
resources/views/welcome.blade.phpのheadタグ内に以下のscriptタグを追加

```
<script src="{{ mix('js/app.js') }}" defer></script>
```

### ブラウザ確認
http://localhost:8880/ でWelcomeページを表示しましょう。  
[DevTools](https://developers.google.com/web/tools/chrome-devtools/)にてConsoleログが確認できる状態にしてください。  


別のブラウザから http://localhost:8880/public-event にアクセスすると、WelcomeページのConsoleログに

```
{message: "PUBLIC", socket: null}
```

が表示されます
