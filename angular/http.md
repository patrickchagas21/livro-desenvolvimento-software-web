# HTTP

Um recurso muito útil no desenvolvimento de software web é consultar fontes de dados. Por exemplo, suponha que um conjunto de objetos esteja presente em um arquivo ou então esteja disponível por meio de uma URL de uma API. Para usar esse rcurso, o Angular fornece o `HttpModule`.

O `HttpModule` é disponibilizado pelo pacote `@angular/http` e é preciso registrá-lo no `AppModule`:

```typescript
import { HttpModule } from '@angular/http';

@NgModule({
    imports : [ BrowserModule, FormsModule, HttpModule ],
    ...
})
export class AppModule { }
```

## Consultando dados de um arquivo .json

Uma fonte de dados pode ser um arquivo .json. Por exemplo \(arquivo `eventos.json`\):

```js
[
    {
        "id": 1, 
        "nome": "XIX Congresso de Computação e Sistemas de Informação", 
        "sigla": "ENCOINFO"
    },
    {
        "id": 2, 
        "nome": "XIII Simpósio Brasileiro de Sistemas de Informação", 
        "sigla": "SBSI"
    },
    {
        "id": 3, 
        "nome": "XXXVII Congresso da Sociedade Brasileira de Computação", 
        "sigla": "CSBC"
    }
]
```

Para consultar o arquivo podemos usar o serviço `Http`, disponibilizado pelo módulo `HttpModule`. Para isso, é necessário, primeiro, injetá-lo no construtor de um serviço:

```typescript
import { Injectable } from '@angular/core';
import { Evento } from './Evento';
import { Http }       from '@angular/http';
import { Observable }     from 'rxjs/Observable';
import 'rxjs/add/operator/map';

@Injectable()
export class EventosService {
    constructor(private http : Http) {
    }

    ...
}
```

Depois, o serviço pode disponibilizar métodos que consultem e retornem o conteúdo do arquivo. Por exemplo, o método `all()`:

```typescript
all() : Observable<Evento[]>{
    return this.http.get('../../public/dados/eventos.json')
        .map(response => response.json() as Evento[]);
}
```

O serviço `Http` fornece o método `get()`, que aceita um caminho \(ou uma URL\) para o arquivo .json \(neste caso `eventos.json`\). O retorno desse método é do tipo `Observable` sobre o qual chamamos a função `map()`, que faz algum processamento sobre o retorno do método anterior e o retorna. 

A sintaxe `response => response.json()` é chamada de lambda ou função seta \(arrow function\) e permite passar uma função como callback para o método `map()`. Neste caso, a função lambda tem um parâmetro, chamado `response`, e o retorno da função é `response.json()`. Por fim, este valor é convertido para `Evento[]` utilizando o operador `as`.

Para usar o serviço, o componente deve interagir com ele conforme um ciclo de vida específico \(e assíncrono\) \(trecho de código a seguir é do componente `EventoManagerComponent`\):

```typescript
ngOnInit(): void {
  this.eventosService.all().subscribe(eventos => this.eventos = eventos);
}
```

Para utilizar o valor de um `Observable` é necessário chamar o método `subscribe()`, que aceita três parâmetros:

* a callback que trata do retorno da `Observable` que opera com sucesso
* a callback que trata do retorno da `Observable` quando ocorre um erro
* a callback que executa quando o processo é concluído

Lidar com esse ciclo de vida é importante porque seu comportamento assíncrono muda a forma de utilizar os métodos -- por exemplo, não há um retorno do método para ser usado diretamente.

## Mais sobre serviços

Esta seção demonstra como criar um serviço para usar o serviço `Http` e consumir uma API.

**Exemplo \(trecho do arquivo src\/app\/eventos.service.ts\):**

```typescript
import { Injectable } from '@angular/core';
import { Http, Response, Headers, RequestOptions } from '@angular/http';
import { Evento } from './evento';
import { Observable } from 'rxjs/Observable';

@Injectable()
export class EventosService {
 constructor(private http: Http) { }
 getEventos(filtro: string) : Observable<Evento[]> { }

 private extractData(res: Response) { }

 private handleError(error: any) { }

 save(evento: Evento) : Observable<Evento> { }

 delete(evento: Evento) : Observable<number> { }
}
```

Note a presença da função `@Injectable()` \(do pacote `@angular/core`\). Essa anotação da classe é importante para informar a outras classes que `EventosService` deve ser utilizada como um serviço.

![Serviço EventosService](uml-eventos-service.png)

No construtor é utilizado o recurso de injeção de dependência para que a classe `EventosService` possa usar a classe `Http` \(fornecida pelo pacote `@angular/http`\), que fornece acesso a recursos de comunicação sobre HTTP utilizando XHR \(XmlHttpRequest\).

Considera-se neste capítulo que a API utilizada tem a seguinte estrutura:

![API](uml-api.png)

As rotas são:

* `GET /eventos`: retorna um objeto `{data:[]}` representando os eventos existentes. Essa rota aceita o parâmetro de URL `q`, que contém uma string usada para filtrar o conjunto de dados
* `GET /eventos/{id}`: retorna um objeto `{data: {}}` representando um evento cujo identificador é representado pelo parâmetro de rota `id`
* `POST /eventos/{id}`: salva \(cadastra ou atualiza\) os dados de um evento cujo identificador é representado pelo parâmetro de rota `id` e os dados estão no corpo do protocolo HTTP. O retorno é um objeto `{data:{}}` com os dados atualizados do evento salvo
* `DELETE /eventos/{id}`: exclui um evento cujo identificador é representado pelo parâmetro de rota `id`. O retorno é um objeto `{data:number}` \(com `1` indicando que o evento foi excluído e `0`, que não foi incluído\)

## Consultando eventos

O método `getEventos(filtro: string): Observable<Evento[]>` utiliza o conceito de "Observable" para consultar dados via HTTP. Pela definição da API, utiliza a rota `GET /eventos`:

```typescript
getEventos(filtro: string) : Observable<Evento[]> {
    return this.http.get(this.apiUrl + '?q=' + filtro)
        .map(this.extractData)
        .catch(this.handleError);
}
```

## Salvando eventos

O método `save(evento: Evento) : Observable<Evento>` utiliza `http.post()` e envia os dados de um evento para a rota `POST /eventos/{id}` da API:

```typescript
save(evento: Evento) : Observable<Evento> {
    let body = JSON.stringify(evento);
    let headers = new Headers({ 'Content-Type': 'application/json' });
    let options = new RequestOptions({ headers: headers });

    return this.http.post(this.apiUrl + evento.id, body, options)
                    .map(this.extractData)
                    .catch(this.handleError);
}
```

Requisições POST precisam de mais informações do que requisições GET. Por isso, o método `http.post()` aceita mais parâmetros:

* `this.apiUrl + evento.id`: representa a URL da requisição
* `body`: representa os dados que estão sendo enviados. O objeto `body` é uma string criada a partir da chamada de `JSON.stringify()` recebendo como parâmetro o objeto `evento` \(o objeto que tem os dados que devem ser salvos\), que é convertido para string
* `options`: é um objeto do tipo `RequestOptions` que é usado para indicar que a requisição utiliza o tipo `application/json`.

## Excluindo eventos

O método `delete(evento: Evento) : Observable<number>` utiliza `http.delete()` e envia o identificador de um evento para a rota `DELETE /eventos/{id}` da API:

```typescript
delete(evento: Evento) : Observable<number> {
    return this.http.delete(this.apiUrl + evento.id)
        .map(this.extractData)
        .catch(this.handleError);
}
```

A estrutura de `http.delete()` é bastante similar à de `http.get()`.

