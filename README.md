# Testes de Performance com JMeter: API de Pets

Desafio prático do módulo de Testes de Performance (PGATS). O plano `api-de-pets.jmx` testa a
[api-de-pets](https://github.com/tiagotaq/api-de-pets) com dois modelos de carga, usando apenas
elementos nativos do JMeter 5.6.3 (nenhum plugin).

## O que o plano faz

Em **cada** Thread Group o usuário virtual executa o mesmo fluxo, com login antes das consultas:

| Passo | Requisição | Validações |
| --- | --- | --- |
| 1 | `POST /login` (uma vez por usuário virtual) | status 200, resposta traz `token` (extraído para `${token}`) |
| 2 | `POST /pets` com nome único | status 201, `name` devolvido igual ao enviado |
| 3 | `GET /pets` | status 200, lista com pelo menos um pet |

Todas as chamadas têm uma asserção de tempo de resposta (`max_ms`, padrão 1000 ms). Entre as
consultas há think time aleatório (500 a 1500 ms). Os usuários vêm de `data/usuarios.csv`
(`tiago` e `ricardo`, os usuários pré-cadastrados da API).

| Thread Group | Modelo | Como a carga é definida |
| --- | --- | --- |
| Modelo FECHADO | Thread Group padrão | 10 usuários virtuais, rampa de 10 s, 5 iterações cada |
| Modelo ABERTO | Open Model Thread Group (nativo desde o JMeter 5.5) | 5 chegadas por segundo durante 30 s, chegadas aleatórias |

No modelo fechado a quantidade de usuários é fixa e a taxa de requisições é consequência do
tempo de resposta. No aberto a taxa de chegada é fixa, independente de a API estar lenta.

Os grupos rodam em sequência. O cronograma do modelo aberto conta a partir do **início do
teste**, por isso ele começa só depois de `open_delay` segundos (35 s, com folga sobre os ~20 s
que o fechado leva) e termina com `open_drain` segundos de pausa para as últimas iterações
acabarem. Se aumentar a carga do fechado, aumente também `open_delay`.

## Como executar

1. Suba a API (`npm install && npm start` na pasta da api-de-pets, porta 3000).
2. Rode em modo linha de comando, que é o recomendado para carga:

```bash
jmeter -n -t api-de-pets.jmx -l results/execucao.jtl -e -o results/relatorio
```

O relatório HTML fica em `results/relatorio/index.html`. A pasta `results/relatorio` precisa não
existir ou estar vazia a cada execução. Também é possível abrir o `.jmx` na GUI do JMeter
(`File > Open`) para ver a estrutura.

## Parâmetros (`-Jnome=valor`)

| Parâmetro | Padrão | Efeito |
| --- | --- | --- |
| `host`, `port` | `localhost`, `3000` | Onde está a API |
| `closed_threads` | 10 | Usuários virtuais do modelo fechado |
| `closed_rampup` | 10 | Rampa em segundos |
| `closed_loops` | 5 | Iterações por usuário |
| `open_delay` | 35 | Segundos até o modelo aberto começar |
| `open_rate` | 5 | Chegadas por segundo |
| `open_duration` | 30 | Duração das chegadas, em segundos |
| `open_drain` | 5 | Pausa final para as iterações terminarem |
| `think_ms`, `think_range_ms` | 500, 1000 | Think time: mínimo e faixa aleatória |
| `max_ms` | 1000 | Limite de tempo de resposta por chamada |

Exemplo, apontando para outro servidor e com mais carga:

```bash
jmeter -n -t api-de-pets.jmx -l results/execucao.jtl -Jhost=192.168.0.10 -Jclosed_threads=30 -Jopen_rate=15
```

## Resultado da execução com os parâmetros padrão

560 amostras, 0 erros. Fechado: 110 chamadas. Aberto: 150 chegadas, 450 chamadas, vazão de
cerca de 5 req/s por endpoint (a taxa de chegada configurada). Tempo médio de 4 ms em `POST /pets`
e `GET /pets`, e de 26 a 30 ms em `POST /login` (a API confere a senha com bcrypt).

## Observações sobre a API

- Os dados ficam em memória. Reinicie a API entre execuções para voltar ao estado inicial.
- `GET /pets` devolve todos os pets, então a resposta cresce a cada `POST /pets`.
- O nome do pet é único por usuário (a API devolve 409 em duplicidade), por isso o plano gera
  `Pet-<thread>-<UUID>` a cada iteração.
