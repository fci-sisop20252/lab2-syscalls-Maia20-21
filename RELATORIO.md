# 📝 Relatório do Laboratório 2 - Chamadas de Sistema

---

## 1️⃣ Exercício 1a - Observação printf() vs 1b - write()

### 💻 Comandos executados:
```bash
strace -e write ./ex1a_printf
strace -e write ./ex1b_write
```

### 🔍 Análise

**1. Quantas syscalls write() cada programa gerou?**
- ex1a_printf: 9 syscalls
- ex1b_write: 7 syscalls

**2. Por que há diferença entre os dois métodos? Consulte o docs/printf_vs_write.md**

A diferença acontece porque "printf()" utiliza um buffer intermediário (a "carteira") antes de enviar os dados para a saída, enquanto "write()" envia diretamente para o sistema operacional (o "banco").  
No "printf()", as mensagens podem ser acumuladas em memória e só são enviadas em determinados momentos: quando o buffer enche, quando há um "\n", quando o programa termina ou quando chamamos "fflush()".  
Já o "write()" realiza a chamada ao sistema operacional imediatamente, a cada execução, sem armazenar em buffer.  
Por isso, "printf()" pode ser mais rápido, com menos chamadas de sistema, enquanto "write()" é mais direto.


**3. Qual método é mais previsível? Por quê você acha isso?**

Como o método "write()" tem uma interação mais direta com o sistema operacional, isso faz com que ele seja mais previsível que o método "printf()".

---

## 2️⃣ Exercício 2 - Leitura de Arquivo

### 📊 Resultados da execução:
- File descriptor: 3
- Bytes lidos: 127

### 🔧 Comando strace:
```bash
strace -e openat,read,close ./ex2_leitura
```

### 🔍 Análise

**1. Qual file descriptor foi usado? Por que não começou em 0, 1 ou 2?**

O file descriptor usado foi o 3. Ele não começou em 0, 1 ou 2 porque esses são reservados por padrão para as entradas e saídas padrões (stdin, stdout e stderr) do processo.

**2. Como você sabe que o arquivo foi lido completamente?**

A leitura foi completa porque o número de bytes lidos, 127, corresponde exatamente ao tamanho do buffer solicitado (BUFFER_SIZE - 1). Sendo assim, o programa exibiu todo o conteúdo do arquivo teste1.txt.

**3. Por que verificar retorno de cada syscall?**

É preciso verificar o retorno de cada syscall para garantir que a operação foi bem-sucedida e tratar possíveis erros. Se uma chamada como open(), read() ou close() falhar, o retorno é um valor negativo como -1, sendo possível identificar qual foi o erro usando perror().

---

## 3️⃣ Exercício 3 - Contador com Loop

### 📋 Resultados (BUFFER_SIZE = 64):
- Linhas: 25 (esperado: 25)
- Caracteres: 1300
- Chamadas read(): 21
- Tempo: 0.000091 segundos

### 🧪 Experimentos com buffer:

| Buffer Size | Chamadas read() | Tempo (s) |
|-------------|-----------------|-----------|
| 16          |        82       |  0.000215 |
| 64          |        21       |  0.000091 |
| 256         |        6        |  0.000083 |
| 1024        |        2        |  0.000058 |

### 🔍 Análise

**1. Como o tamanho do buffer afeta o número de syscalls?**

O tamanho do buffer afeta o número de syscalls necessárias para ler um arquivo inteiro. Como é possível ver nos resultados, quanto maior o tamanho do buffer, menor o número de chamadas.

**2. Todas as chamadas read() retornaram BUFFER_SIZE bytes? Discorra brevemente sobre**

Não, nem todas as chamadas read() retornaram o número de bytes igual ao BUFFER_SIZE solicitado. A última chamada de read antes de retornar 0 (fim do arquivo) sempre retorna uma quantidade de bytes menor que o tamanho do buffer, pois o restante do arquivo é menor que o buffer. Por exemplo, com o buffer de 16 bytes, a última leitura retornou 4 bytes, e com o buffer de 1024 bytes, a última leitura retornou 276 bytes.

**3. Qual é a relação entre syscalls e performance?**

Quanto menor o número de chamadas de sistema (syscalls), maior a performance. Como as syscalls são operações que exigem que o sistema transite do modo de usuário para o modo kernel, esse processo acaba consumindo tempo e recursos, fazendo com que o tempo de execução diminuia drasticamente à medida que o números de chamadas read() cai. Ou seja, isso demonstra que minimizar as syscalls lendo blocos de dados maiores de uma só vez, melhora o desempenho geral do programa.

---

## 4️⃣ Exercício 4 - Cópia de Arquivo

### 📈 Resultados:
- Bytes copiados: 1364
- Operações: 6
- Tempo: 0.000211 segundos
- Throughput: 6312.94 KB/s

### ✅ Verificação:
```bash
diff dados/origem.txt dados/destino.txt
```
Resultado: [X] Idênticos [ ] Diferentes

### 🔍 Análise

**1. Por que devemos verificar que bytes_escritos == bytes_lidos?**

Verificar que bytes_escritos == bytes_lidos é importante na hora da cópia, pois, assim, garante que não houve perda da informação. Se o valor de retorno de write() for menor do que o número de bytes que se tentou escrever (bytes_lidos), isso indica que a escrita falhou parcialmente.

**2. Que flags são essenciais no open() do destino?**

Flags essenciais no open() do arquivo de destino:
- O_WRONLY: indica que o arquivo será aberto apenas para escrita.
- O_CREAT: cria o arquivo de destino.
- O_TRUNC: trunca o arquivo para o tamanho zero se ele já existir, garantindo que não haja resíduos de dados de uma execução anterior.

**3. O número de reads e writes é igual? Por quê?**

Sim, o strace mostra 6 chamadas de read e 6 chamadas de write, ou seja, é igual. Isso acontece porque o programa lê um bloco de dados do arquivo de origem e, imediatamente, tenta escrever esse mesmo bloco no arquivo de destino dentro do mesmo loop.

**4. Como você saberia se o disco ficou cheio?**

O disco cheio é um erro que seria detectado pela função write(). Se o disco ficar sem espaço durante a operação, a chamada write() retornaria um valor menor que 0 e a variável errno seria definida para indicar o erro específico (exemplo: ENOSPC). A verificação bytes_escritos != bytes_lidos também pegaria essa falha.

**5. O que acontece se esquecer de fechar os arquivos?**

É possível que cause um problema chamado "vazamento de file descriptor" (file descriptor leak). Cada processo tem um número limitado de file descriptors que pode usar, se o programa não os libera, eles continuam ocupados e, com o tempo, pode eventualmente esgotar os recursos disponíveis, impedindo que o programa abra novos arquivos.

---

## 🎯 Análise Geral

### 📖 Conceitos Fundamentais

**1. Como as syscalls demonstram a transição usuário → kernel?**

Essa transição feitas pelas syscalls é demonstrado através do strace, que mostra o momento exato em que essa transição ocorre, exibindo as chamadas de sistema como read() e write(), que são os pontos de entrada para o kernel.

**2. Qual é o seu entendimento sobre a importância dos file descriptors?**

O sistema operacional usa os File descriptors para identificar recursos de I/O, atuando como um ponteiro ou identificador para um arquivo aberto. A importância deles é que eles abstraem a complexidade do I/O, permitindo que os programas manipulem diferentes tipos de recursos da mesma maneira, usando as mesmas syscalls como read(), write() e close().

**3. Discorra sobre a relação entre o tamanho do buffer e performance:**

Essa relação é crucial, pois, como as chamadas de sistema são uma operação custosa devido à transição de contexto entre modos usuário e kernel, ler e escrever dados em blocos (buffers) maiores, reduz o número de syscalls necessárias para processar um arquivo inteiro. Isso acaba diminuindo a sobrecarga e resulta em um tempo de execução menor.

### ⚡ Comparação de Performance

```bash
# Teste seu programa vs cp do sistema
time ./ex4_copia
time cp dados/origem.txt dados/destino_cp.txt
```

**Qual foi mais rápido?** ./ex4_copia

**Por que você acha que foi mais rápido?**

O programa que executou ./ex4_copia foi mais rápido que o cp, pois o cp do sistema é uma ferramenta com muito mais funcionalidades e etapas, etapas essas necessárias para uma cópia byte a byte.

---

## 📤 Entrega
Certifique-se de ter:
- [ ] Todos os códigos com TODOs completados
- [ ] Traces salvos em `traces/`
- [ ] Este relatório preenchido como `RELATORIO.md`

```bash
strace -e write -o traces/ex1a_trace.txt ./ex1a_printf
strace -e write -o traces/ex1b_trace.txt ./ex1b_write
strace -o traces/ex2_trace.txt ./ex2_leitura
strace -c -o traces/ex3_stats.txt ./ex3_contador
strace -o traces/ex4_trace.txt ./ex4_copia
```
# Bom trabalho!
