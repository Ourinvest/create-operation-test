# Extrato de Operações Cambiais
Estamos sempre procurando desenvolvedores para auxiliar em nossa transformação digital, este teste tem a missão de fornecer um pequeno desafio que possa ser resolvido de inúmeras maneiras utilizando métodos diferentes e que possa ser utilizado durante uma entrevista técnica. Vamos trabalhar com arquivos .json que contêm informações sobre operações de câmbio. Sua tarefa é criar um sistema orientado a objetos que leia um arquivo de entrada, processe as operações descritas nele e escreva os resultados em outro arquivo .json.  
  
Os arquivos .json de entrada terão a seguinte estrutura: 

```json
{ 
  "balance":"float",  // saldo em reais do cliente 
  "limit":"float",     // limite em reais do quanto o cliente pode operar em um período 
  "operations":[ 
    { 
      "type":"string",    // Se a operação é de envio(OUT) ou de recebimento(IN) 
      "spot":"float",     // valor da moeda no mercado 
      "spread":"float",   // valor do percentual 
      "fx_quantity":"float",  // quantidade de moeda estrangeira sendo negociada
      "created_at":"string"   // horário da execução da operação em UTC escrita segundo ISO 8601
    }
  ] 
}
``` 
O objeto em questão lida com duas partes:
- Informações do clientes, neste caso, saldo (balance) e limite (limit).
- Uma lista de tamanho irrestrito de operações contendo as minimas informações necessárias para a criação do extrato.

Sua tarefa é processar estas operações de câmbio. Existem dois tipos de operações: "In", que significa que o cliente está recebendo dinheiro, e "Out", que significa que o cliente está enviando dinheiro. Para cada operação, você precisa calcular o valor em reais correspondente, levando em conta o valor do 'spot' (paridade da moeda estrangeira com o real) e o 'spread' (um percentual adicional cobrado sobre a operação). 

Seu código deve respeitar a ordem das operações conforme fornecida no arquivo de entrada. Se uma operação não puder ser executada por algum motivo (por exemplo, se exceder o limite do cliente), ela deve ser removida do arquivo de saída. O máximo de operações devem ser processadas. 

O arquivo de saída deve ter a seguinte estrutura: 

```json
{ 
  "balance":"float",    // saldo atualizado em reais do cliente 
  "limit":"float",       // limite atualizado em reais do cliente, 
  "operations":[ 
    { 
       "real_quantity":"float",  // quantidade em reais do quanto foi negociado 
       "created_at":"string"   // horário da execução da operação em UTC escrita segundo ISO 8601
    } 
 ] 
}
```
## Conceitos

- `spot`: Podemos entender como o preço de compra ou venda da moeda em questão no momento de uma operação (claramente um resumo para a execução da tarefa), um exemplo facil é entender o dolar, hoje na cotação de 4,79 BRL. No exercicio, quando pedimos `fx_quantity: 1000` significa que estamos operando 1000 USD, o que resultaria em uma operação sem `spread` de 4790 BRL.
- `spread`: Trata-se da diferença entre o valor pago por uma instituição financeira ou banco na compra de dinheiro e o valor recebido por ele na venda, empréstimo ou operação feita com esse dinheiro. É tratado diferentemente em casos de In/Out, quando estamos recebendo (In) o banco reduz este pequeno custo do valor total que receberiamos, e quando estamos enviando (Out) é cobrado como taxa adicional.

## Algumas regras adicionais: 

- Para operações de entrada ("In"), o valor em reais é calculado como `Valor De Moeda Estrangeira * (1 - spread) * spot`. Para operações de saída ("Out"), o valor em reais é `Valor De Moeda Estrangeira * (1 + spread) * spot`. 
- O saldo do cliente é aumentado para operações de entrada e diminuído para operações de saída, podendo ficar negativo.
- O limite do cliente é sempre subtraído pelo valor em reais da operação. Nunca podendo ser menor que 0.

## Exemplos
### Caso 1 - Mais simples possível

Entrada:
```
{
  "balance": 10000.0,
  "limit": 10000.0,
  "operations": [{"type": "In", "spot": 1.0, "spread": 0.0, "fx_quantity": 5000.0, "created_at": "2023-07-19T21:07:22.556467"},
                 {"type": "Out", "spot": 1.0, "spread": 0.0, "fx_quantity": 5000.0, "created_at": "2023-07-20T21:07:22.556467"}]
}
```

Extrato:
Saldo   | Limite | Valor Utilizado na operação | Explicação
--------- | ------ | ------ | ------
10000 | 10000 | Inicio Extrato |
15000 | 5000 | + 5000 | Operação IN (entrando dinheiro) soma o saldo, mas toda operação trava limite
10000 | 0 | - 5000 | Operação OUT (entrando dinheiro) subtrai o saldo, mas toda operação trava limite
10000 | 0 | Finalizado Extrato | Saldo não movimentou devido a entrada e saída, porém o limite foi todo consumido

### Caso 2 - Operação irregular entre operações regulares
Entrada:
```
{
  "balance": 10000.0,
  "limit": 10000.0,
  "operations": [{"type": "In", "spot": 1.0, "spread": 0.0, "fx_quantity": 5000.0, "created_at": "2023-07-19T21:07:22.556467"},
                 {"type": "Out", "spot": 1.0, "spread": 0.0, "fx_quantity": 6000.0, "created_at": "2023-07-20T21:07:22.556467"},
                 {"type": "In", "spot": 1.0, "spread": 0.0, "fx_quantity": 5000.0, "created_at": "2023-07-20T21:07:22.559999"}]
}
```

Extrato:
Saldo   | Limite | Valor Utilizado na operação | Explicação
--------- | ------ | ------ | ------
10000 | 10000 | Inicio Extrato |
15000 | 5000 | + 5000 | Operação IN (entrando dinheiro) soma o saldo, mas toda operação trava limite
15000 | 5000 |  | Operação OUT de 6000 não aprovada devido a falta de limite
20000 | 0 | + 5000 | Operação IN (entrando dinheiro) soma o saldo, mas toda operação trava limite
20000 | 0 | Finalizado Extrato | Saldo aumentou em 10000 devido as duas operações de entrada, porém o limite foi todo consumido

### Caso 3 - Sequencia de operações irregulares com conceito de spot adicionado
Entrada:
```
{
  "balance": 10000.0,
  "limit": 10000.0,
  "operations": [{"type": "In", "spot": 2.0, "spread": 0.0, "fx_quantity": 5000.0, "created_at": "2023-07-19T21:07:22.556467"},
                 {"type": "Out", "spot": 2.0, "spread": 0.0, "fx_quantity": 6000.0, "created_at": "2023-07-20T21:07:22.556467"},
                 {"type": "In", "spot": 2.0, "spread": 0.0, "fx_quantity": 5000.0, "created_at": "2023-07-20T21:07:22.559999"}]
}
```

Extrato:
Saldo   | Limite | Valor Utilizado na operação | Explicação
--------- | ------ | ------ | ------
10000 | 10000 | Inicio Extrato |
20000 | 5000 | + 10000 | Operação IN é dobrada devido ao spot
20000 | 5000 |  | Operação OUT de 12000 não aprovada devido a falta de limite
20000 | 5000 |  | Operação IN de 10000 não aprovada devido a falta de limite
20000 | 5000 | Finalizado Extrato | Saldo aumentou em 10000 devido a operação de entrada, porém o limite foi todo consumido

## Ao codificar sua solução, tenha em mente as seguintes diretrizes: 

1. Seu código deve ser conciso e expressivo. 
2. Deve haver uma separação clara de conceitos e adaptabilidade. 
3. O sistema deve ser orientado a objetos. 
4. Devem ser aplicados conceitos básicos de design e arquitetura de código. 
5. As partes críticas do código devem ser testadas e testáveis. 
6. O sistema não deve manter estado entre as execuções. 
7. A documentação do código deve ser clara e acessível. 

Se você puder usar conteinerização, documentar adequadamente objetos de valor e entidades agregadas (DDD), documentar os critérios de aceitação e usar um gerenciador de pacotes, isso será considerado um diferencial. 

Estamos ansiosos para ver o que você pode fazer!
