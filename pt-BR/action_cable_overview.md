**NÃO LEIA ESTE ARQUIVO NO GITHUB, OS GUIAS SÃO PUBLICADOS NO https://guiarails.com.br.**
**DO NOT READ THIS FILE ON GITHUB, GUIDES ARE PUBLISHED ON https://guides.rubyonrails.org.**

Action Cable Overview
=====================

Neste guia, você irá aprender como *Action Cable* funciona e como usar *WebSockets*
para incorporar funcionalidades de tempo real em sua aplicação Rails.

Ao ler este guia você aprenderá:

* O que é um *Action Cable* e sua integração *backend* e *frontend*
* Como configurar um *Action Cable*
* Como configurar canais
* Configuração de *Deployment* e Arquitetura para rodar a *Action Cable*

--------------------------------------------------------------------------------

O que é o *Action Cable*?
---------------------

O *Action Cable* integra-se perfeitamente [WebSockets](https://pt.wikipedia.org/wiki/WebSocket) com o resto da sua aplicação Rails. Permite que recursos em tempo real sejam escritos em Ruby no mesmo estilo e forma que o resto de sua aplicação Rails, ao mesmo tempo em que possui desempenho e escabilidade. Isso é uma oferta _full-stack_ que fornece um _framework_ Javascript do lado do cliente (_client-side_) e um _framework_ Ruby do lado do servidor (_server-side_). Você tem acesso ao seu _model_ de domínio completo escrito com o *Active Record* ou o ORM de sua escolha.

Terminologia
-----------

O *Action Cable* utiliza *WebSockets* ao invés do protocolo de requisição-resposta HTTP.
Tanto o *Action Cable* quanto os *WebSockets* apresentam uma terminologia menos familiar:

### Conexões

Conexões formam a base do relacionamento cliente-servidor.
Um único servidor *Action Cable* pode lidar com várias instâncias de conexão. Ele possui uma instância de conexão para cada conexão via *WebSocket*. Um único usuário pode ter vários *WebSockets* abertos para sua aplicação se ele utilizar várias abas do navegador ou dispositivos.

### Consumidores

O client de uma conexão *WebSocket* é chamado de *consumidor*. No *Action Cable*, o consumidor é criado pelo framework JavaScript do lado do cliente.

### Canais

Cada consumidor pode, por sua vez, se inscrever em vários *canais*.
Cada canal encapsula uma unidade lógica de trabalho, semelhante ao que um
*controller* faz em uma configuração MVC regular.
Por exemplo, você pode ter um *ChatChannel* e um *AppearancesChannel*, e um
consumidor pode ser inscrito em um ou em ambos os canais.
Um consumidor deve se inscrever em, pelo menos, um canal.

### Assinantes

Quando o consumidor está inscrito em um canal, ele age como um *assinante*.
A conexão entre o assinante e o canal é, adivinhe, chamada de assinatura.
Um consumidor pode atuar como um assinante de um determinado canal qualquer
número de vezes. Por exemplo, um consumidor pode se inscrever em várias salas de
chat ao mesmo tempo. (E lembre-se que um usuário físico pode ter vários
consumidores, um por aba/dispositivo aberto para sua conexão).

__Pub/Sub_
---------------

_[Pub/Sub](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)_, ou
_Publish-Subscribe_, refere-se a um paradigma de fila de mensageria o qual os
remetentes de uma informação (_publishers_) enviam dados à uma classe abstrata de
destinatários (_subscribers_) sem especificar um destinatário individual.
*Action Cable* utiliza essa abordagem para manter a comunicação entre o servidor
e diversos clientes.

### **Broadcastings**

Uma transmissão (*broadcasting*) é um link *pub/sub* em que qualquer coisa transmitida pela emissora é
enviada diretamente para os assinantes (*subscribers*) do canal que estão transmitindo essa transmissão.
Cada canal pode transmitir zero ou mais transmissões.

## Componentes _Server-Side_

### *Connections*

Para cada _WebSocket_ aceito pelo servidor, um objeto *connection* é instanciado. Esse objeto
se torna o pai de todos os *channel subscriptions* que são criados dali pra frente.
A *connection* em si não lida com nenhuma lógica específica da aplicação além da
autenticação e autorização. O cliente de um _WebSocket *connection*_ é chamado de
*consumer*. Um usuário individual criará um par de *consumer-connection* para cada
aba do navegador, janela ou dispositivo que ele tiver aberto.

*Connections* são instâncias de `ApplicationCable::Connection`, que estendem de
[`ActionCable::Connection::Base`][]. Em `ApplicationCable::Connection`, você
autoriza a *connection* recebida e procede para estabelecê-la, caso o
usuário possa ser identificado.

#### Configuração de uma *Connection*

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    private
      def find_verified_user
        if verified_user = User.find_by(id: cookies.encrypted[:user_id])
          verified_user
        else
          reject_unauthorized_connection
        end
      end
  end
end
```

Aqui, [`identified_by`][] designa um identificador de *connection* que pode ser usado para
encontrar uma *connection* específica mais tarde. Note que qualquer coisa marcada
como um identificador criará automaticamente um *delegate* pelo mesmo nome em
qualquer instância de *channel* criada a partir da *connection*.

Esse exemplo se baseia no fato de que você já lidou a autenticação do usuário em
algum outro lugar na sua aplicação e essa autenticação bem sucedida definiu um
*cookie* criptografado com o ID do usuário.

O *cookie* é então enviado automaticamente para a instância da *connection* quando há
a tentativa de criar uma nova *connection*, e você o usa para definir o `current_user`.
Ao identificar a *connection* para o mesmo usuário, você também garante que você pode retornar todas as *connections* em aberto para um usuário específico (e potencialmente desconectá-los, caso o usuário seja deletado ou desautorizado).

Se a sua abordagem de autenticação inclui o uso de uma sessão (*session*), você usa o armazenamento com _cookies_ para a
sessão, seu _cookie_ de sessão é denominado `_session` e a chave de ID do usuário é `user_id` você
pode usar esta abordagem:

```ruby
verified_user = User.find_by(id: cookies.encrypted['_session']['user_id'])
```

[`ActionCable::Connection::Base`]: https://api.rubyonrails.org/classes/ActionCable/Connection/Base.html
[`identified_by`]: https://api.rubyonrails.org/classes/ActionCable/Connection/Identification/ClassMethods.html#method-i-identified_by

#### Lidando com Exceções (*Exceptions*)

Por padrão, exceções não tratadas são capturadas e registradas no *logger* do Rails. Se você gostaria de
interceptar globalmente essas exceções e relatá-las a um serviço externo de rastreamento de *bugs*, por
exemplo, você pode fazer isso com [`rescue_from`] []:

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    rescue_from StandardError, with: :report_error

    private

    def report_error(e)
      SomeExternalBugtrackingService.notify(e)
    end
  end
end
```

[`rescue_from`]: https://api.rubyonrails.org/classes/ActiveSupport/Rescuable/ClassMethods.html#method-i-rescue_from

### *Channels*

O *channel* encapsula uma unidade lógica de trabalho, parecido com o que um
*controller* faz em um *MVC* comum. Por padrão, o *Rails* cria uma classe pai
`ApplicationCable::Channel` (que estende [`ActionCable::Channel::Base`][]) para encapsular a lógica compartilhada entre seus
*channels*.

#### Configuração do *Channel* pai

```ruby
# app/channels/application_cable/channel.rb
module ApplicationCable
  class Channel < ActionCable::Channel::Base
  end
end
```

Então, você criaria suas próprias classes de *channel*. Por exemplo, você poderia ter um
`ChatChannel` e um `AppearanceChannel`:

```ruby
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
end
```

```ruby
# app/channels/appearance_channel.rb
class AppearanceChannel < ApplicationCable::Channel
end
```

[`ActionCable::Channel::Base`]: https://api.rubyonrails.org/classes/ActionCable/Channel/Base.html

Um *consumer* poderia então ser inscrito para qualquer ou ambos os *channels*.

#### *Subscriptions*

*Consumers* se inscrevem a *channels*, agindo como *subscribers*. A *connection*
deles é chamada de *subscription*. Mensagens produzidas são então roteadas para esses
*channel subscriptions* baseados em um identificador enviado pelo *channel consumer*.

```ruby
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  # Chamado quando o *consumer* tornou-se um *subscriber*
  # desse *channel* com sucesso.
  def subscribed
  end
end
```

#### Lidando com Exceções

Tal como acontece com `ApplicationCable::Connection`, você também pode usar [`rescue_from`] [] em um
canal específico para lidar com exceções levantadas:

```ruby
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  rescue_from 'MyError', with: :deliver_error_message

  private

  def deliver_error_message(e)
    broadcast_to(...)
  end
end
```

## Componentes _Client-Side_

### Conexões

Consumidores precisam de uma instância da conexão do seu lado. Esta conexão pode
ser estabelecida usando o seguinte JavaScript, que é gerado por padrão pelo
Rails:

#### Conectar o Consumidor

```js
// app/javascript/channels/consumer.js
// Action Cable provides the framework to deal with WebSockets in Rails.
// You can generate new channels where WebSocket features live using the `bin/rails generate channel` command.

import { createConsumer } from "@rails/actioncable"

export default createConsumer()
```

Isto vai preparar um consumidor que conectará em `/cable` em seu servidor por
padrão. A conexão não vai ser estabelecida até que você também tenha
especificado ao menos uma inscrição que você tem interesse em ter.

O consumidor pode optar receber um argumento que especifica a _URL_ para se
conectar. Ela pode ser uma _string_, ou uma função que retorna uma _string_ que
vai ser chamada quando o _WebSocket_ é aberto.

```js
// Especifica uma _URL_ diferente para se conectar
createConsumer('https://ws.example.com/cable')

// Utiliza uma função para gerar a _URL_ dinamicamente
createConsumer(getWebSocketURL)

function getWebSocketURL {
  const token = localStorage.get('auth-token')
  return `https://ws.example.com/cable?token=${token}`
}
```

#### Assinante

Um consumidor se torna um assinante criando uma assinatura para um canal:

```js
// app/javascript/channels/chat_channel.js
import consumer from "./consumer"

consumer.subscriptions.create({ channel: "ChatChannel", room: "Best Room" })

// app/javascript/channels/appearance_channel.js
import consumer from "./consumer"

consumer.subscriptions.create({ channel: "AppearanceChannel" })
```

Enquanto isto cria uma assinatura, a funcionalidade necessária para responder
aos dados recebidos será descrita mais tarde.

Um consumidor pode agir como um assinante para um dado canal qualquer número de
vezes. Por exemplo, um consumidor pode assinar várias salas de _chat_ ao mesmo
tempo.

```js
// app/javascript/channels/chat_channel.js
import consumer from "./consumer"

consumer.subscriptions.create({ channel: "ChatChannel", room: "1st Room" })
consumer.subscriptions.create({ channel: "ChatChannel", room: "2nd Room" })
```

## Interações Cliente-Servidor

### *Streams*

*Streams* fornecem o mecanismo por onde os *channels* direcionam o conteúdo publicado (*broadcasts*) para seus assinantes. Por exemplo, o código a seguir usa [`stream_from`][] para se inscrever na transmissão (*broadcasting*) chamada `chat_Best Room` quando o valor do parâmetro `:room` é `"Best Room"`:

```ruby
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  def subscribed
    stream_from "chat_#{params[:room]}"
  end
end
```

Então, de outro lugar em sua aplicação Rails, é possível transmitir para essa `room` chamando [`broadcast`][]:

```ruby
ActionCable.server.broadcast("chat_Best Room", { body: "This Room is Best Room." })
```

Se você possui um *stream* que está relacionada a uma *model*, então o nome da transmissão pode ser gerada a partir do *channel* e *model*. Por exemplo, o código a seguir usa [`stream_for`][] para se inscrever em uma transmissão como `comments:Z2lkOi8vVGVzdEFwcC9Qb3N0LzE`, onde `Z2lkOi8vVGVzdEFwcC9Qb3N0LzE` corresponde ao ID Global da *model* Post.

```ruby
class CommentsChannel < ApplicationCable::Channel
  def subscribed
    post = Post.find(params[:id])
    stream_for post
  end
end
```

Você pode agora transmitir para esse *channel* chamando [`broadcast_to`][]:

```ruby
CommentsChannel.broadcast_to(@post, @comment)
```

[`broadcast`]: https://api.rubyonrails.org/classes/ActionCable/Server/Broadcasting.html#method-i-broadcast
[`broadcast_to`]: https://api.rubyonrails.org/classes/ActionCable/Channel/Broadcasting/ClassMethods.html#method-i-broadcast_to
[`stream_for`]: https://api.rubyonrails.org/classes/ActionCable/Channel/Streams.html#method-i-stream_for
[`stream_from`]: https://api.rubyonrails.org/classes/ActionCable/Channel/Streams.html#method-i-stream_from

### *Broadcastings*

Um *broadcasting* é um link *pub/sub* em que qualquer coisa transmitida por um *publisher* é encaminhada diretamente para os assinantes do *channel*, este, que está transmitindo o *broadcasting* de mesmo nome. Cada *channel* pode estar transmitindo zero ou mais *broadcastings*.

*Broadcastings* são puramente filas de espera online e dependentes de tempo. Se um consumidor não estiver transmitindo (assinante de um determinado *channel*), ele não vai receber o *broadcast* caso se conecte mais tarde.

### *Subscriptions*

Quando um consumidor está inscrito em um *channel*, ele age como assinante (*subscriber*). Essa conexão é chamada de assinatura (*subscription*). Mensagens recebidas são então direcionadas para esses inscritos do *channel* baseadas em um identificador enviado pelo *cable consumer*

```js
// app/javascript/channels/chat_channel.js
// Assumindo que você já requeriu os direitos para enviar notificações web
import consumer from "./consumer"

consumer.subscriptions.create({ channel: "ChatChannel", room: "Best Room" }, {
  received(data) {
    this.appendLine(data)
  },

  appendLine(data) {
    const html = this.createLine(data)
    const element = document.querySelector("[data-chat-room='Best Room']")
    element.insertAdjacentHTML("beforeend", html)
  },

  createLine(data) {
    return `
      <article class="chat-line">
        <span class="speaker">${data["sent_by"]}</span>
        <span class="body">${data["body"]}</span>
      </article>
    `
  }
})
```

### Passando Parâmetros para *Channel*

Você pode passar parâmetros do lado do cliente para o lado do servidor quando cria a *subscription*. Por exemplo:

```ruby
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  def subscribed
    stream_from "chat_#{params[:room]}"
  end
end
```

Um objeto passado como primeiro argumento em `subscriptions.create` torna-se a *hash* `params` em *cable channel*. A *keyword* `channel` é obrigatória:

```js
// app/javascript/channels/chat_channel.js
import consumer from "./consumer"

consumer.subscriptions.create({ channel: "ChatChannel", room: "Best Room" }, {
  received(data) {
    this.appendLine(data)
  },

  appendLine(data) {
    const html = this.createLine(data)
    const element = document.querySelector("[data-chat-room='Best Room']")
    element.insertAdjacentHTML("beforeend", html)
  },

  createLine(data) {
    return `
      <article class="chat-line">
        <span class="speaker">${data["sent_by"]}</span>
        <span class="body">${data["body"]}</span>
      </article>
    `
  }
})
```

```ruby
# Em algum lugar do seu app isso pode ser chamado, talvez,
# por um novo NewCommentJob.
ActionCable.server.broadcast(
  "chat_#{room}",
  {
    sent_by: 'Paul',
    body: 'This is a cool chat app.'
  }
)
```

### Retransmitindo uma Mensagem

Um caso de uso comum é ter que retransmitir uma mensagem enviada por um cliente para qualquer outro cliente conectado

```ruby
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  def subscribed
    stream_from "chat_#{params[:room]}"
  end

  def receive(data)
    ActionCable.server.broadcast("chat_#{params[:room]}", data)
  end
end
```

```js
// app/javascript/channels/chat_channel.js
import consumer from "./consumer"

const chatChannel = consumer.subscriptions.create({ channel: "ChatChannel", room: "Best Room" }, {
  received(data) {
    // data => { sent_by: "Paul", body: "This is a cool chat app." }
  }
}

chatChannel.send({ sent_by: "Paul", body: "This is a cool chat app." })
```

A retransmissão vai ser recebida por todos os clientes conectados, _incluindo_ o cliente que enviou a mensagem. Note que `params` são os mesmos de quando você se inscreveu no *channel*

## Full-Stack Examples

The following setup steps are common to both examples:

  1. [Setup your connection](#componentes-server-side-connections).
  2. [Setup your parent channel](#configuracao-do-channel-pai).
  3. [Connect your consumer](#conectar-o-consumidor).

### Example 1: User Appearances

Here's a simple example of a channel that tracks whether a user is online or not
and what page they're on. (This is useful for creating presence features like showing
a green dot next to a user name if they're online).

Create the server-side appearance channel:

```ruby
# app/channels/appearance_channel.rb
class AppearanceChannel < ApplicationCable::Channel
  def subscribed
    current_user.appear
  end

  def unsubscribed
    current_user.disappear
  end

  def appear(data)
    current_user.appear(on: data['appearing_on'])
  end

  def away
    current_user.away
  end
end
```

When a subscription is initiated the `subscribed` callback gets fired and we
take that opportunity to say "the current user has indeed appeared". That
appear/disappear API could be backed by Redis, a database, or whatever else.

Create the client-side appearance channel subscription:

```js
// app/javascript/channels/appearance_channel.js
import consumer from "./consumer"

consumer.subscriptions.create("AppearanceChannel", {
  // Called once when the subscription is created.
  initialized() {
    this.update = this.update.bind(this)
  },

  // Called when the subscription is ready for use on the server.
  connected() {
    this.install()
    this.update()
  },

  // Called when the WebSocket connection is closed.
  disconnected() {
    this.uninstall()
  },

  // Called when the subscription is rejected by the server.
  rejected() {
    this.uninstall()
  },

  update() {
    this.documentIsActive ? this.appear() : this.away()
  },

  appear() {
    // Calls `AppearanceChannel#appear(data)` on the server.
    this.perform("appear", { appearing_on: this.appearingOn })
  },

  away() {
    // Calls `AppearanceChannel#away` on the server.
    this.perform("away")
  },

  install() {
    window.addEventListener("focus", this.update)
    window.addEventListener("blur", this.update)
    document.addEventListener("turbolinks:load", this.update)
    document.addEventListener("visibilitychange", this.update)
  },

  uninstall() {
    window.removeEventListener("focus", this.update)
    window.removeEventListener("blur", this.update)
    document.removeEventListener("turbolinks:load", this.update)
    document.removeEventListener("visibilitychange", this.update)
  },

  get documentIsActive() {
    return document.visibilityState == "visible" && document.hasFocus()
  },

  get appearingOn() {
    const element = document.querySelector("[data-appearing-on]")
    return element ? element.getAttribute("data-appearing-on") : null
  }
})
```

##### Client-Server Interaction

1. **Client** connects to the **Server** via `App.cable =
ActionCable.createConsumer("ws://cable.example.com")`. (`cable.js`). The
**Server** identifies this connection by `current_user`.

2. **Client** subscribes to the appearance channel via
`consumer.subscriptions.create({ channel: "AppearanceChannel" })`. (`appearance_channel.js`)

3. **Server** recognizes a new subscription has been initiated for the
appearance channel and runs its `subscribed` callback, calling the `appear`
method on `current_user`. (`appearance_channel.rb`)

4. **Client** recognizes that a subscription has been established and calls
`connected` (`appearance_channel.js`) which in turn calls `install` and `appear`.
`appear` calls `AppearanceChannel#appear(data)` on the server, and supplies a
data hash of `{ appearing_on: this.appearingOn }`. This is
possible because the server-side channel instance automatically exposes all
public methods declared on the class (minus the callbacks), so that these can be
reached as remote procedure calls via a subscription's `perform` method.

5. **Server** receives the request for the `appear` action on the appearance
channel for the connection identified by `current_user`
(`appearance_channel.rb`). **Server** retrieves the data with the
`:appearing_on` key from the data hash and sets it as the value for the `:on`
key being passed to `current_user.appear`.

### Example 2: Receiving New Web Notifications

The appearance example was all about exposing server functionality to
client-side invocation over the WebSocket connection. But the great thing
about WebSockets is that it's a two-way street. So now let's show an example
where the server invokes an action on the client.

This is a web notification channel that allows you to trigger client-side
web notifications when you broadcast to the right streams:

Create the server-side web notifications channel:

```ruby
# app/channels/web_notifications_channel.rb
class WebNotificationsChannel < ApplicationCable::Channel
  def subscribed
    stream_for current_user
  end
end
```

Create the client-side web notifications channel subscription:

```js
// app/javascript/channels/web_notifications_channel.js
// Client-side which assumes you've already requested
// the right to send web notifications.
import consumer from "./consumer"

consumer.subscriptions.create("WebNotificationsChannel", {
  received(data) {
    new Notification(data["title"], body: data["body"])
  }
})
```

Broadcast content to a web notification channel instance from elsewhere in your
application:

```ruby
# Somewhere in your app this is called, perhaps from a NewCommentJob
WebNotificationsChannel.broadcast_to(
  current_user,
  title: 'New things!',
  body: 'All the news fit to print'
)
```

The `WebNotificationsChannel.broadcast_to` call places a message in the current
subscription adapter's pubsub queue under a separate broadcasting name for each
user. For a user with an ID of 1, the broadcasting name would be
`web_notifications:1`.

The channel has been instructed to stream everything that arrives at
`web_notifications:1` directly to the client by invoking the `received`
callback. The data passed as argument is the hash sent as the second parameter
to the server-side broadcast call, JSON encoded for the trip across the wire
and unpacked for the data argument arriving as `received`.

### More Complete Examples

See the [rails/actioncable-examples](https://github.com/rails/actioncable-examples)
repository for a full example of how to set up Action Cable in a Rails app and adding channels.

## Configuração

O _Action Cable_ tem duas configurações necessárias: um adaptador de assinatura (_subscription_) e origens de requisição (_request_) permitidas.

### Adaptador de assinatura

Por padrão, _Action Cable_ procura um arquivo de configuração em `config/cable.yml`.
O arquivo deve especificar um adaptador (_adapter_) para cada ambiente Rails. Veja o
[Dependências](#dependencias) seção para obter informações adicionais sobre adaptadores.

```yaml
development:
  adapter: async

test:
  adapter: async

production:
  adapter: redis
  url: redis://10.10.3.153:6381
  channel_prefix: appname_production
```
#### Configuração do Adaptador (_Adapter_)

Abaixo está uma lista dos adaptadores de assinatura disponíveis para usuários finais.

##### Adaptador Async

O adaptador assíncrono destina-se ao desenvolvimento / teste e não deve ser usado em produção.

##### Adaptador Redis

O adaptador Redis requer que os usuários forneçam uma URL apontando para o servidor Redis.
Além disso, um `channel_prefix` pode ser fornecido para evitar colisões de nome de canal
ao usar o mesmo servidor Redis para vários aplicativos. Veja a
[Documentação Redis PubSub](https://redis.io/topics/pubsub#database-amp-scoping) para mais detalhes.

##### Adaptador PostgreSQL

O adaptador PostgreSQL usa o _pool_ de conexão do _Active Record_ e, portanto, o
configuração do banco de dados `config/database.yml` do aplicativo, para sua conexão.
Isso pode mudar no futuro. [#27214](https://github.com/rails/rails/issues/27214)

### Origens de Requisição Permitidas

Action Cable só aceitará requisições de origens especificadas, que são
passado para a configuração do servidor como um array. As origens podem ser instâncias de
_strings_ ou expressões regulares, contra as quais uma verificação de correspondência será realizada.

```ruby
config.action_cable.allowed_request_origins = ['https://rubyonrails.com', %r{http://ruby.*}]
```

Para desativar e permitir requisições de qualquer origem:

```ruby
config.action_cable.disable_request_forgery_protection = true
```

Por padrão, o _Action Cable_ permite todas as requisições de localhost:3000 durante a execução
no ambiente de desenvolvimento.

### Configuração do Consumidor

Para configurar a URL, adicione uma chamada para [`action_cable_meta_tag`][] em seu _layout_ HTML
HEAD. Isso usa uma URL ou caminho (_path_) normalmente definido via `config.action_cable.url` nos
arquivos de configuração de ambiente.

[`action_cable_meta_tag`]: https://api.rubyonrails.org/classes/ActionCable/Helpers/ActionCableHelper.html#method-i-action_cable_meta_tag

### Configuração do _Worker Pool_

O _pool_ de _workers_ é usado para executar retornos (_callbacks_) de conexão e ações de _channel_ em
isolamento da _thread_ principal do servidor. _Action Cable_ permite que a aplicação
configure o número de _threads_ processados ​​simultaneamente no _worker pool_.

```ruby
config.action_cable.worker_pool_size = 4
```

Além disso, observe que seu servidor deve fornecer pelo menos o mesmo número de conexões de banco de dados
que você tem de _workers_. O tamanho do _worker pool_ de trabalho padrão é definido como 4, então
isso significa que você deve disponibilizar pelo menos 4 conexões de banco de dados.
Você pode mudar isso em `config/database.yml` através do atributo `pool`.

### Log no lado do client

O log *client side* é desabilitado por padrão. Você pode habilitar essa configuração em `ActionCable.logger.enable` trocando para `true`.

```ruby
import * as ActionCable from '@rails/actioncable'

ActionCable.logger.enabled = true
```

### Outras Configurações

A outra opção comum de configurar são as _tags_ de _log_ aplicadas ao
_logger_ por conexão. Aqui está um exemplo que usa
o ID da conta do usuário, se disponível, senão "sem conta" durante a marcação:

```ruby
config.action_cable.log_tags = [
  -> request { request.env['user_account_id'] || "no-account" },
  :action_cable,
  -> request { request.uuid }
]
```

Para obter uma lista completa de todas as opções de configuração, consulte o
classe `ActionCable::Server::Configuration`.

## Executando Servidores _Cable_ Autônomos

### Na Aplicação

O _Action Cable_ pode ser executado junto com sua aplicação Rails. Por exemplo, para
escutar as requisições _WebSocket_ em `/websocket`, especifique esse caminho para
`config.action_cable.mount_path`:

```ruby
# config/application.rb
class Application < Rails::Application
  config.action_cable.mount_path = '/websocket'
end
```

Você pode usar `ActionCable.createConsumer()` para conectar ao
_cable server_ se `action_cable_meta_tag` for invocado no layout. Caso contrário, um caminho é
especificado como primeiro argumento para `createConsumer` (e.g. `ActionCable.createConsumer("/websocket")`).

Para cada instância de seu servidor que você cria e para cada _worker_ que seu servidor
instancia, você também terá uma nova instância do _Action Cable_, mas o uso do Redis
mantém as mensagens sincronizadas entre as conexões.

### Autônomo

Os _cable servers_ a cabo podem ser separados do servidor de aplicação normal. Este
ainda é uma aplicação Rack, mas é sua própria aplicação Rack. O recomendado
para a configuração básica é a seguinte:

```ruby
# cable/config.ru
require_relative "../config/environment"
Rails.application.eager_load!

run ActionCable.server
```

Então você inicia o servidor usando um _binstub_ em `bin/cable` ala:

```
#!/bin/bash
bundle exec puma -p 28080 cable/config.ru
```

O código irá iniciar um _cable server_ na porta 28080.

### Notas

O servidor WebSocket não tem acesso à sessão, mas tem
acesso aos cookies. Isso pode ser usado quando você precisa lidar com
autenticação. Você pode ver uma maneira de fazer isso com o Devise neste [artigo](https://greg.molnar.io/blog/actioncable-devise-authentication/).

## Dependências

O _Action Cable_ fornece uma interface de adaptador de assinatura para processar seus
_pubsub_ internos. Por padrão, adaptadores assíncronos, _inline_, PostgreSQL e Redis
estão incluídos. O adaptador padrão
em novas aplicações Rails é o adaptador assíncrono (`async`).

O lado Ruby das coisas é construído em cima de [websocket-driver](https://github.com/faye/websocket-driver-ruby),
[nio4r](https://github.com/celluloid/nio4r) e [concurrent-ruby](https://github.com/ruby-concurrency/concurrent-ruby).

## Implantação

O _Action Cable_ é alimentado por uma combinação de _WebSockets_ e _threads_. Tanto o
[_plumbing_](https://www.techopedia.com/definition/31509/plumbing) do _framework_ e o trabalho do _channel_ especificado pelo usuário são tratados internamente,
usando suporte de _thread_ nativo do Ruby. Isso significa que você pode usar todos os seus
*models* dos Rails sem problemas, contanto que você não tenha cometido nenhum pecado de _thread-safety_.

O servidor _Action Cable_ implementa o _Rack socket hijacking API_,
permitindo assim o uso de um padrão _multithread_ para o gerenciamento de conexões
internamente, independentemente de o servidor de aplicativos ser multiencadeado ou não.

Assim, _Action Cable_ funciona com servidores populares como _Unicorn_, _Puma_ e
_Passenger_.

## Teste

Você pode encontrar instruções detalhadas de como testar a sua funcionalidade *Action Cable* no
[guia de teste](testing.html#testing-action-cable).
