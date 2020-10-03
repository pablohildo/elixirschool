---
version: 1.0.1
title: OTP Distribution
---

## Introdução a Distribuição
Nós podemos rodar nossas aplicações Elixir em um conjunto de diferentes nós distribuídos através de um único hospedeiro ou múltiplos hospedeiros.
Elixir permite que nos comuniquemos através desses nós via alguns mecanismos diferentes que vamos contornar nessa lição.

{% include toc.html %}

## Comunicação Entre Nós

Elixir roda na Erlang VM, o que significa que tem acesso à poderosa [funcionalidade de distribuição](http://erlang.org/doc/reference_manual/distributed.html) do Erlang.

> Um sistema distribuído Erlang consiste de um número de sistemas Erlang em execução se comunicando uns com os outros.
Cada um desses sistemas em execução é chamado de nó.

Um nó é qualquer sistema Erlang em execução que recebeu um nome.
Nós podemos iniciar um nó abrindo uma sessão no `iex` e nomeando-o:

```bash
iex --sname alex@localhost
iex(alex@localhost)>
```

Vamos abrir um outro nó em outra janela do terminal:

```bash
iex --sname kate@localhost
iex(kate@localhost)>
```

Esses dois nós podem mandar mensagens um ao outro usando `Node.spawn_link/2`.

### Comunicando-se com `Node.spawn_link/2`

Essa função recebe dois parâmetros:
* O nome do nó para qual você quer se conectar
* A função a ser executada pelo processo remoto rodando naquele nó

Ela estabelece a conexão ao nó remoto e executa a função dada naquele nó, retornando o PID do processo relacionado.

Vamos definir um módulo, `Kate`, no nó `kate` que sabe como introduzir Kate, a pessoa:

```elixir
iex(kate@localhost)> defmodule Kate do
...(kate@localhost)>   def say_name do
...(kate@localhost)>     IO.puts "Hi, my name is Kate"
...(kate@localhost)>   end
...(kate@localhost)> end
```

#### Enviando Mensagens

Agora, nós podemos usar [`Node.spawn_link/2`](https://hexdocs.pm/elixir/Node.html#spawn_link/2) para fazer o nó `alex` perguntar ao nó `kate` para chamar a função `say_name`: 
```elixir
iex(alex@localhost)> Node.spawn_link(:kate@localhost, fn -> Kate.say_name end)
Hi, my name is Kate
#PID<10507.132.0>
```

#### Uma Nota Sobre E/S e Nós

Note que, apesar de `Kate.say_name/0` ser executada no nó remoto, é o nó local, ou o que chamou, que recebe a saída de `IO.puts`.
Isso é porque o nó local é o **líder do grupo**.
A Erlang VM gerencia E/S através de processos.
Isso permite que executemos tarefas de E/S, como `IO.puts`, através de nós distribuídos.
Esses processos distribuídos são geridos pelo líder do grupo do processo de E/S.
O líder do grupo é sempre o nó que gera o processo.
Então, já que nosso nó `alex` é de onde chamamos `spawn_link/2`, esse nó é o líder do grupo e a saída de `IO.puts` vai ser direcionada à stream de saída daquele nó.

#### Respondendo a mensagens

E se nós quisermos que o nó que recebe a mensagem envie alguma *resposta* de volta ao remetente? Nós podemos usar uma simples configuração de `receive/1` e [`send/3`](https://hexdocs.pm/elixir/Process.html#send/3) para atingir exatamente isso.

Nós vamos fazer com que nosso nó `alex` gere uma conexão ao nó `kate` e dê ao nó `kate` uma função anônima pra executar.
Essa função anônima vai aguardar o recebimento de uma tupla em particular descrevendo uma mensagem e o PID do nó `alex`.
Ele vai responder a essa mensagem enviando uma mensagem de volta ao PID do nó `alex`, usando `send`:

```elixir
iex(alex@localhost)> pid = Node.spawn_link :kate@localhost, fn ->
...(alex@localhost)>   receive do
...(alex@localhost)>     {:hi, alex_node_pid} -> send alex_node_pid, :sup?
...(alex@localhost)>   end
...(alex@localhost)> end
#PID<10467.112.0>
iex(alex@localhost)> pid
#PID<10467.112.0>
iex(alex@localhost)> send(pid, {:hi, self()})
{:hi, #PID<0.106.0>}
iex(alex@localhost)> flush()
:sup?
:ok
```

#### Uma Nota Sobre Comunicação Entre Nós em Diferentes Redes

Se você quiser enviar mensagens entre nós de diferentes redes, nós devemos iniciar os nós nomeados com um cookie compartilhado:

```bash
iex --sname alex@localhost --cookie secret_token
```

```bash
iex --sname kate@localhost --cookie secret_token
```

Apenas nós iniciados com o mesmo `cookie` vão ser capazes de se conectar com sucesso um ao outro.

#### Limitações de `Node.spawn_link/2`

Enquanto `Node.spawn_link/2` ilustra o relacionamento entre nós e a forma como podemos mandar mensagens entre eles, ela _não_ é realmente a escolha certa para uma aplicação que vai ser executada através de nós distribuídos.
`Node.spawn_link/2` gera processos em isolamento, ou seja, processos que não são supervisionados.
Se ao menos houvesse uma forma de gerar processos supervisionados e assíncronos _através de nós_...
If only there was a way to spawn supervised, asynchronous processes _across nodes_...

## Tarefas Distribuídas

[Tarefas Distribuídas](https://hexdocs.pm/elixir/master/Task.html#module-distributed-tasks) permitem que geremos tarefas supervisionadas através de nós.
Nós vamos construir uma aplicação supervisora simples que alavanga tarefas distribuídas para permitir que usuários conversem entre si através de uma sessão no `iex`, através de nós distribuídos.

### Definindo a Aplicação Supervisora

Gere sua aplicação:

```
mix new chat --sup
```

### Adicionando o Supervisor de Tarefas à Árvore de Supervisão

Um Supervisor de Tarefas dinamicamente supervisiona tarefas.
Ele é iniciado sem filhos, costumeiramente _sob_ um supervisor próprio, e pode ser usado mais tarde para supervisionar qualquer número de tarefas.

Nós vamos adicionar um Supervisor de Tarefas à árvore de supervisão de nossa aplicação e chamá-lo de `Chat.TaskSupervisor`.

```elixir
# lib/chat/application.ex
defmodule Chat.Application do
  @moduledoc false

  use Application

  def start(_type, _args) do
    children = [
      {Task.Supervisor, name: Chat.TaskSupervisor}
    ]

    opts = [strategy: :one_for_one, name: Chat.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

Agora nós sabemos que onde quer que nossa aplicação seja iniciada em um dado nó, o `Chat.Supervisor` estará rodando e pronto para supervisionar tarefas.

### Enviando Mensagens com Tarefas Supervisionadas

Nós vamos iniciar tarefas supervisionadas com a função [`Task.Supervisor.async/5`](https://hexdocs.pm/elixir/master/Task.Supervisor.html#async/5).

Essa função deve receber quatro parâmetros:

* O supervisor que queremos usar para supervisionar a tarefa. 
* Isso pode ser passado como uma tupla de `{SupervisorName, remote_node_name}` para podermos supervisionar a tarefa no nó remoto.
* O nome do módulo em que queremos executar a função.
* O nome da função que queremos executar.
* Quaisquer parâmetros que precisarem ser passados para esta função.

Você pode passar um quinto parâmetro opcional descrevendo opções de desligamento.
Não iremos nos preocupar com isso aqui.

Nossa aplicação de Chat é bem simples.
Ela envia mensagens a nós remotos e nós remotos respondem a essas mensagens usando `IO.puts` para expô-las à STDOUT do nó remoto.

Primeiro, vamos definir uma função, `Chat.receive_message/1`, que queremos que nossa tarefa execute em um nó remoto.

```elixir
# lib/chat.ex
defmodule Chat do
  def receive_message(message) do
    IO.puts message
  end
end
```

Depois disso, vamos ensinar o módulo `Chat` a enviar a mensagem para um nó remoto usando uma tarefa supervisionada.
Nós vamos definir um método `Chat.send_message/2` que vai direcionar esse processo:

```elixir
# lib/chat.ex
defmodule Chat do
  ...

  def send_message(recipient, message) do
    spawn_task(__MODULE__, :receive_message, recipient, [message])
  end

  def spawn_task(module, fun, recipient, args) do
    recipient
    |> remote_supervisor()
    |> Task.Supervisor.async(module, fun, args)
    |> Task.await()
  end

  defp remote_supervisor(recipient) do
    {Chat.TaskSupervisor, recipient}
  end
end
```

Vamos ver isso em ação.

Em uma janela de terminal, inicie nossa aplicação de chat em uma sessão `iex` nomeada.

```bash
iex --sname alex@localhost -S mix
```

Abra outra janela de terminal para iniciar a aplicação em um nó nomeado diferente:

```bash
iex --sname kate@localhost -S mix
```

Agora, no nó `alex`, nós podemos enviar uma mensagem para o nó `kate`:

```elixir
iex(alex@localhost)> Chat.send_message(:kate@localhost, "hi")
:ok
```

Mude para a janela `kate` e você deveria ver a mensagem:

```elixir
iex(kate@localhost)> hi
```

O nó `kate` pode responder de volta ao nó `alex`:

```elixir
iex(kate@localhost)> hi
Chat.send_message(:alex@localhost, "how are you?")
:ok
iex(kate@localhost)>
```

E isso vai aparecer na sessão do `iex` do nó `alex`::

```elixir
iex(alex@localhost)> how are you?
```

Vamos revisitar nosso código e entender o que está acontecendo aqui.

Nós temos uma função `Chat.send_message/2` que recebe o nome do nó remoto em que desejamos rodar nossas tarefas supervisionadas e a mensagem que queremos enviar para aquele nó.

Essa função chama nossa função `spawn_task/4` que começa uma tarefa assíncrona rodando o nó remoto com o nome dado, supervisionada pelo `Chat.TaskSupervisor` neste nó remoto.
Nós sabemos que o Supervisor de Tarefas com nome `Chat.TaskSupervisor` está rodando naquele nó porque aquele nó _também_ está rodando uma instância de nossa aplicação Chat e o `Chat.TaskSupervisor` é iniciado como parte da árvore de supervisão do aplicativo de Chat.

Nós estamos dizendo ao `Chat.TaskSupervisor` para supervisionar uma tarefa que execute a função `Chat.receive_message` com um parâmetro de qualquer mensagem que foi passada a `spawn_task/4` por `send_message/2`.

Então, `Chat.receive_message("hi")` é chamada no nó remoto `kate`, fazendo com que a mensagem `"hi"` seja colocada no stream STDOUT daquele nó.
Nesse caso, já que a tarefa está sendo supervisionada no nó remoto, este nó é o gerente do grupo para esse processo de E/S.

### Respondendo a Mensagens de Nós Remotos

Vamos tornar nossa aplicação de Chat um pouco mais inteligente.
Até então, qualquer número de usuários pode rodar a aplicação em uma sessão `iex` nomeada e começar a bater papo.
Mas digamos que existe um cachorro branco de tamanho médio chamado Moebi que não quer ficar de fora. Moebi quer ser incluído na aplicação de Chat mas infelizmente ele não sabe digitar, porque ele é um cachorro.
Então, nós ensinaremos nosso módulo `Chat` a responder a quaisquer mensagens enviadas a um nó chamado `moebi@localhost` por Moebi.
Não importa o que você dizer para Moebi, ele vai responder com `"chicken?"` porque a única coisa que ele deseja é comer frango.

Nós vamos definir outra versão de nossa função `send_message/2` que faz pattern matching no parâmetro `recipient`
Se o destinatário é `:moebi@localhost`, nós vamos:

* Pegar o nome do nó atual usando `Node.self()`
* Dar o nome do nó atual, ou seja, o remetente, para uma nova função `receive_message_for_moebi/2`, para que nós póssamos enviar uma mensagem _de volta_ para este nó.

```elixir
# lib/chat.ex
...
def send_message(:moebi@localhost, message) do
  spawn_task(__MODULE__, :receive_message_for_moebi, :moebi@localhost, [message, Node.self()])
end
```

Depois disso, nós vamos definir uma função `receive_message_for_moebi/2` que usa `IO.puts` para exibir a mensagem no stream STDOUT do nó `moebi` _e_ envia uma mensagem de volta para o remente:

```elixir
# lib/chat.ex
...
def receive_message_for_moebi(message, from) do
  IO.puts message
  send_message(from, "chicken?")
end
```

Ao chamar `send_message/2` com o nome do nó que enviou a mensagem original (o "nó remetente"), nós estamos dizendo ao nó _remoto_ para gerar uma tarefa supervisionada de volta naquele nó remetente.

Vamos ver isso em ação.
Em três diferentes janelas de terminal, abra três nós diferentes nomeados:

```bash
iex --sname alex@localhost -S mix
```

```bash
iex --sname kate@localhost -S mix
```

```bash
iex --sname moebi@localhost -S mix
```

Vamos fazer `alex` enviar uma mensagem para `moebi`:

```elixir
iex(alex@localhost)> Chat.send_message(:moebi@localhost, "hi")
chicken?
:ok
```

Nós podemos ver que o nó `alex` recebeu uma resposta, `"chicken?"`.
Se nós abrirmos o nó `kate`, veremos que nenhuma mensagem foi recebida, já que nem `alex` nem `moebi` enviaram nada para ela (desculpa, `kate`).
E se nós abrirmos a janela do terminal do nó `moebi`, nós vamos ver ver a mensagem que o nó `alex` enviou:

```elixir
iex(moebi@localhost)> hi
```

## Testando Código Distribuído

Vamos começar escrevendo um teste simples para nossa função `send_message`.

```elixir
# test/chat_test.ex
defmodule ChatTest do
  use ExUnit.Case, async: true
  doctest Chat

  test "send_message" do
    assert Chat.send_message(:moebi@localhost, "hi") == :ok
  end
end
```

Se rodarmos nossos testes com `mix test`, veremos que vão falhar com o seguinte erro:

```elixir
** (exit) exited in: GenServer.call({Chat.TaskSupervisor, :moebi@localhost}, {:start_task, [#PID<0.158.0>, :monitor, {:sophie@localhost, #PID<0.158.0>}, {Chat, :receive_message_for_moebi, ["hi", :sophie@localhost]}], :temporary, nil}, :infinity)
         ** (EXIT) no connection to moebi@localhost
```

Esse erro faz todo sentido--nós não podemos conectar a um nó chamado `moebi@localhost` porque esse nó não está rodando.

Nós podemos fazer esse teste passar realizando as seguintes etapas:

* Abrir uma outra janela do terminal e rodar o nó nomeado: `iex --sname moebi@localhost -S mix`.
* Rodar os testes no primeiro terminal através de um nó que roda os testes do mix em uma sessão `iex`: `iex --sname sophie@localhost -S mix test`.

Isso é muito trabalhoso e definitivamente não seria considerado um processo automatizado de teste.

Existem duas abordagens diferentes que podemos seguir aqui:

1.
Excluir condicionalmente testes que precisam de nós distribuídos, se o nó necessário não estiver rodando.

2.
Configurar nossa aplicação para evitar gerar tarefas em nós remotos no ambiente de teste.

Vamos dar uma olhada na primeira abordagem.

### Excluindo Condicionalmente Testes com Tags

Nós vamos adicionar uma tag `ExUnit` para esse teste:

```elixir
#test/chat_test.ex
defmodule ChatTest do
  use ExUnit.Case, async: true
  doctest Chat

  @tag :distributed
  test "send_message" do
    assert Chat.send_message(:moebi@localhost, "hi") == :ok
  end
end
```

E nós vamos adicional um pouco de lógica condicional ao nosso test helper para excluir testes com essas tags se os testes _não_ estiverem rodando em um nó nomeado.

```elixir
exclude =
  if Node.alive?, do: [], else: [distributed: true]

ExUnit.start(exclude: exclude)
```

Nós vamos checar para ver se o nó está vivo, ou seja, se o nó é parte de um sistema distribuído com [`Node.alive?`](https://hexdocs.pm/elixir/Node.html#alive?/0).
Se não, nós podemos dizer ao `ExUnit` para pular quaisquer testes com a tag `distributed: true`.
Caso contrário, nós vamos dizê-lo para não excluir nenhum teste.

Agora, se nós rodarmos o bom e velho `mix test`, nós vamos ver:

```bash
mix test
Excluding tags: [distributed: true]

Finished in 0.02 seconds
1 test, 0 failures, 1 excluded
```

E se nós quisermos rodar nossos testes distribuídos, nós simplesmente precisamos seguir as etapas abordadas na seção prévia: rodar o nó `moebi@localhost` _e_ rodar os testes em um nó nomeado através do `iex`.

Vamos dar uma olhada em nossa outra abordagem de testes--configurando a aplicação para se comportar diferentemente em ambientes diferentes.

### Configuração Específica para Ambientes da Aplicação

A parte do nosso código que diz ao `Task.Supervisor` para iniciar uma outra tarefa supervisionada em um nó remoto é aqui:

```elixir
# app/chat.ex
def spawn_task(module, fun, recipient, args) do
  recipient
  |> remote_supervisor()
  |> Task.Supervisor.async(module, fun, args)
  |> Task.await()
end

defp remote_supervisor(recipient) do
  {Chat.TaskSupervisor, recipient}
end
```

`Task.Supervisor.async/5` pega um primeiro parâmetro do supervisor que queremos usar.
Se nós passarmos a dupla de `{SupervisorName, location}`, ela vai inicializar o supervisor dado no nó remoto dado.
De qualquer forma, se passarmos um primeiro parâmetro de um nome de um supervisor para `Task.Supervisor`, ele vai utilizar esse supervisor para supervisionar a tarefa localmente.

Vamos tornar a função `remote_supervisor/1` configurável baseada no ambiente.
No ambiente de desenvolvimento, ela vai retornar `{Chat.TaskSupervisor, recipient}` e no ambiente de teste ela vai retornar `Chat.TaskSupervisor`.

Nós faremos isso através de variáveis da aplicação.

Crie um arquivo, `config/dev.exs`, e adicione:

```elixir
# config/dev.exs
use Mix.Config
config :chat, remote_supervisor: fn(recipient) -> {Chat.TaskSupervisor, recipient} end
```

Crie um arquivo, `config/test.exs` e adicione:

```elixir
# config/test.exs
use Mix.Config
config :chat, remote_supervisor: fn(_recipient) -> Chat.TaskSupervisor end
```

Lembre de descomentar essa linha no `config/config.exs`:

```elixir
use Mix.Config
import_config "#{Mix.env()}.exs"
```

Por fim, nós vamos atualizar nossa função `Chat.remote_supervisor/1` para procurar e usar a função armazenada em nossa nova variável da aplicação:

```elixir
# lib/chat.ex
defp remote_supervisor(recipient) do
  Application.get_env(:chat, :remote_supervisor).(recipient)
end
```

## Conclusão

As capacidades nativas de distribuição de Elixir, que as possui graças ao poder da Erlang VVM, é um dos recursos que a torna uma ferramenta tão poderosa.
Nós podemos imaginar alavancar a habilidade de Elixir de lidar com computação distribuida para rodar tarefas concorrentes em segundo plano, para suportar aplicações de alta performance, para executar operações caras--o que você quiser.

Essa lição nos dá uma introdução básica ao conceito de distribuição em Elixir e te dá as ferramentas para começar a construir aplicações distribuídas.
Ao usar tarefas supervisionadas, você pode enviar mensagens através de vários nós de uma aplicação distribuída.
