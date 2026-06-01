# OpenMP

## Por que utilizar programação paralela?

Os dois principais motivos:

- **Reduzir o tempo** necessário para solucionar um problema.
- **Resolver problemas mais complexos** e de maior dimensão.

## Modelos de Programação Paralela

### Programação em memória compartilhada (OpenMP, Cilk, CUDA)

- Programação usando processos ou threads.
- Decomposição do domínio ou funcional com granularidade fina, média ou grossa.
- Comunicação atráves de uma **memória compartilhada**.
- Sincronização através de mecanismos de exclusão mútua.

### Programação em memória distribuída (MPI)

- Programação usando processos distribuídos.
- Decomposição do domínio com granularidade grossa.
- Comunicação e sincronização por **troca de mensagens**.

## Introdução do OpenMP

Tipos e protótipos de funções no arquivo:

```c
#include <omp.h>
```

A maioria das contruções OpenMP são diretivas de compilação

```c
# pragma omp construct [clause[clause]...]

// Exemplo:

# pragma omp parallel private(var1, var2) shared(var3, var4)
{
    
}
```

> Em C, `#pragma` é uma diretiva de pré-processamento que modifica/controla alguns comportamentos do compilador que não fazem parte da linguagem em si. De forma resumida, passa comandos especiais ao compilador.

Notas de compilação com **GCC**:

```bash
gcc code.c -o code.out -fopenmp
```

### Funções OpenMP

Algumas funções comuns OpenMP:

```c
// Arquivo de interface da biblioteca OpenMP
#include <omp.h>

// Retorna o identificador da thread (ID)
int omp_get_thread_num();

// Indica o número de threads a executar na região paralela
void omp_set_num_threads(int num_threads);

// Retorna o número de threads que estão executando no momento (por padrão é o número de core do processador)
// Podemos alterar esse padrão usando: export OMP_NUM_THREADS=4 (por exemplo)
int omp_get_num_threads();
```

### Diretivas OpenMP

Algumas diretivas comuns OpenMP:

```c
// Cria uma região paralela. Define variáveis privadas e compartilhas entre as threads
// Variável privada é diferente para cada thread
#pragma omp parallel private(...) shared(...)
{ // precisa do bloco na linha debaixo
    
    // Nesse caso abaixo, apenas a thread mais rápida executa ele
    #pragma omp single
}
```

#### Cláusula shared

Especifica um conjunto de variáveis que são compartilhadas entre as threads (por padrão, as variáveis são shared ao entrar na região paralela).

#### Cláusula private

Especifica um conjunto de variáveis privadas. Os valores no início da região paralela são indefinidos. Ao final da região paralela, as variáveis também são desfeitas, por exemplo, o valor que ela assume ao sair da região paralela pode não ser o mesmo que ela tinha na região paralela.

### Exercício 01: Hello World em OpenMP

```c
#include <stdio.h>
#include <omp.h>

int main() {
    int thread_id, num_threads;
    #pragma omp parallel private(thread_id) shared(num_threads)
    {
        thread_id = omp_get_thread_num();
        #pragma omp single
        num_threads = omp_get_num_threads();
        printf("%d of %d - hello world!\n", thread_id, num_threads);
    }
    return 0;
}

// Para compilar: gcc code01.c -o code.out -fopenmp
```

No OpenMP, podemos usar `#pragma omp for` que permite que usemos laços de repetição sem se preocupar com definir um chunk, init e end para o vetor igual no **MPI**.

### Interações das Threads

OpenMP é um modelo de _multithreading_ de memória compartilhada, portanto, threads se comunicam através de variáveis compartilhadas.

- Compartilhamento não intencional de dados causa **condição de corrida** (quando a saída do programa muda quando as threads são escalonadas de forma diferente)
- O problema ocorre quando duas ou mais threads tentam acessar/alterar o conteúdo da mesma estrutura
- Precisamos pensar numa forma de acesso aos dados para evitar a sincronização (é cara)

### Sincronização

Assegue que uma ou mais threads estão em um estado bem definido em um ponto conhecido da execução. As duas formas mais comuns são:

- **Barreira**: cada thread espera na barreira até a chegada de todas as demais
- **Exclusão mútua**: define um bloco de código onde apenas uma thread pode executar por vez

#### Barrier

```c
#pragma omp parallel
{
    int id = omp_get_thread_num();
    A[id] = big_calc(id);

    #pragma omp barrier
    B[id] = big_calc2(id, A);
}
```

#### Exclusão mútua (Critical)

- **Critical**: apenas uma thread pode entrar por vez

```c
#pragma omp parallel
{
    float b;
    int i, id, nthreads, N;
    id = omp_get_thread_num();
    nthreads = omp_get_num_threads();

    for (i = id; i < N; i += nthreads) {
        b = big_job(i);
        #pragma omp critical
        res += consume(b);
    }
}
```

- **Atomic**: provê exclusão mútua para operações específicas (atribuições, incrementos e decrementos)

```c
#pragma omp parallel
{
    double tmp, b, x = 0;
    b = do_it();
    tmp = big_ugly(b);
    #pragma omp atomic
    x += tmp;
}
```

### Exercício 02: Vector SUM

```c
#include <stdio.h>
#include <omp.h>

#define TAM 1000

int main() {

    int id, nthreads, v[TAM];
    int sum = 0;
    int sum_local = 0;
    
    for (int i = 0; i < TAM; i++)
        v[i] = 1;
    
    #pragma omp parallel private(id) shared(nthreads, sum)
    {
        #pragma omp single
        nthreads = omp_get_num_threads();

        #pragma omp for
        for (int i = 0; i < TAM; i++)
            sum_local += v[i];

        #pragma omp atomic
        sum += sum_local;
    }
    printf("SUM = %d\n", sum);
    return 0;
}
```

### Redução

Combinação de variáveis locais de uma thread em uma variável única. Essa situação se chama **redução**.

Diretiva reduction: `reduction(op: list_vars)`

Dentro de uma região paralela ou divisão de trabalho:

- Será feita uma cópia local de cada variável na lista
- Será inicializada dependendo da operação (ex: 0 para +, 1 para *)
- Atualizações acontecem na cópia local
- Cópias locais são "reduzidas" para uma única variável original (global)

```c
// Exemplo:
#pragma omp for reduction(+: sum)
```

Solução do **Vector SUM (ex 02)** com Reduction:

```c
#include <stdio.h>
#include <omp.h>

#define TAM 100000

int main() {
    int sum = 0, v[TAM];
    for (int i = 0; i < TAM; i++)
        v[i] = 1;

    #pragma omp parallel for reduction(+: sum)
    for (int i = 0; i < TAM; i++)
        sum += v[i];

    printf("SUM = %d\n", sum);
    return 0;
}
```

### Sections

No contexto de programação paralela com OpenMP, `#pragma omp section` é uma diretiva de "compartilhamento de trabalho" (worksharing) usada dentro de um bloco `#pragma omp sections`. Ela divide tarefas distintas e independentes entre as threads, permitindo que cada bloco de código seja executado apenas uma vez por uma única thread, paralelizando funções diferentes:

```c
#include <stdio.h>
#include <omp.h>

int main() {
    #pragma omp parallel 
    // Poderia ser também:  #pragma omp parallel sections
    {
        #pragma omp sections
        {
            #pragma omp section
            {
                // Só 1 thread pega pra fazer essa função
                printf("Função 1\n");
            }
            #pragma omp section
            {
                // Só 1 thread pega pra fazer essa função
                printf("Função 2\n");
            }
        }
        // Todas as threads executam essa função
        printf("Função executada por todas threads!\n");
    }
    return 0;
}
```
