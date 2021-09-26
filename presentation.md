# Sobre o padrão de projeto Singleton em ambientes de concorrência

Desde a apresentação do Singleton pela GoF em meados dos anos 90 muita coisa mudou. Agora temos ambientes que são altamente concorrentes e que podem causar problemas com aquela implementação básica que nos foi passada lá em 1994. Apesar disso, muitos desenvolvedores ainda utilizam a versão `by the book` sem criticar as oportunidades e desafios que esse padrão nos apresenta. Neste contexto, apresentarei algumas reflexões e soluções utilizadas por mim em experiências passadas na utilização desse padrão, bem como a apresentação dos possíveis problemas que uma aplicação concorrente nos coloca.

## Padrão Criacional - Singleton

### Que problema resolve?

- Utilizamos o Singleton quando precisamos garantir que teremos uma única instância em memória de uma dada classe.

```java
// nao podemos ficar invocando `new ` se queremos garantir sempre a mesma instância!
Singleton singleton = new Singleton();
```

- Para garantir uma única instância precisamos criar um ponto de acesso global que possamos alcançar essa instância de qualquer outro artefato! Conseguimos fazer isso com código estático e colocando um construtor privado.

```java
public class Singleton {

    private static Singleton instance;

    // estatico pra ser um ponto de acesso único a partir de qualquer artefato
    public static Singleton getInstance() {
        if (Objects.isNull(instance)) {
            instance = new Singleton();
        }
        return instance;
    }

}

Singleton singleton = Singleton.getInstance();
// usage

```

### Por quê?

- Podemos usar esse padrão com classes que gerenciem recursos escassos, para garantir um `single source of truth` e para reutilizar invocações / criação de instâncias que sejam custosas.

### Versões possíveis de um singleton

#### Versão com carga por demanda, não `thread safe`.

```java
public class DbConnetion {
  private static DbConnetion instance;
  
  private DbConnetion() { }
  
  public static DbConnetion getInstance() {
    if (instance == null) {
      instance = new DbConnetion();
    }
    return instance;
  }
}

//usage
DbConnetion connection = DbConnetion.getInstance();
```

Essa versão é a mais simples, mas apresenta problemas com concorrência. Se duas threads concorrentes tentarem pedir uma instância e `instance` estiver `null`, existe uma possibilidade de acabarmos com duas instâncias em memória. Se estamos falando de um artefato que gerencia recursos escasso, possivelmente teremos um problema intermitente na mão que será difícil de reproduzir.

```java

//thread 1
DbConnetion connection = DbConnetion.getInstance();

//thread 2
DbConnetion connection = DbConnetion.getInstance();

// possivelmente teremos problemas!
```

#### Versão com carga por demanda, `thread safe`.

```java
public class DbConnetion {
  private static DbConnetion instance;
  
  private DbConnetion() { }
  
  // synchronized cria um semáforo por baixo dos panos pra gente
  synchronized public static DbConnetion getInstance() {
    if (instance == null) {
      instance = new DbConnetion();
    }
    return instance;
  }

}

```

Essa versão possui carga por demanda e é segura contra concorrência com o uso da palavra reservada `synchronized`. Mas não sem um custo, cada invocação da função `getInstance` esbarra em um semáforo que tem um `over head` na chamada. Uma quantidade muito grande de invocações dessa chamada podem criar gargalos em algumas chamadas de nossa aplicação.

```java

//thread 1
DbConnetion connection = DbConnetion.getInstance();

//thread 2
DbConnetion connection = DbConnetion.getInstance();

// não teremos problemas aqui (exceto o overhead a cada invocação de singleton)!
```

#### Versão com carga durante startup da aplicação, `thread safe`.

```java
public class DbConnetion {

  public static DbConnetion instance;
  
  private DbConnetion() { }

  static {
    instance = new DbConnetion();
  }

  public static DbConnetion getInstance() {
    return instance;
  }

}

```

Essa versão garante invocações seguras mesmo em ambiente concorrente (como aplicações web por exemplo) e dispensa o custo amortizado da invocação por demanda, uma vez acontece durante a criação da classe pela JVM no startup da aplicação.

### Vantagens que o singleton nos apresenta

- ponto central de chamada para que sempre acessemos a mesma instância de um artefato, ideal para gerenciar recursos escassos ou custosos de manter em memória;

### Desvantagens que o singleton nos apresenta

- é dificil testar o singleton, por ser estático, modificar valores na instância pode vazar conteúdo e sujar o contexto de outros testes. Além de possivelmente quebrar testes rodando concorrente;

### Vantagens e Desvantagens de cada implementação possível

- quando `thread safe` por demanda, evita a carga prematura (talvez desnessária) de um objeto que pode jamais ser utilizado durante a execução da aplicação evitando assim desperdício de memória. Mas, faz isso ao custo de `over head` de sincronização dos semáforos;

- quando não `thread safe` por demanda, também evita a carga prematura e não possuí o custo de sincronização. Se na sua aplicação não fizer diferença a possibilidade de uma segunda instância temporaria, ela pode ser uma opção viável. Porém, muito desaconselhável porquê a medida que o software cresce essa premissa de `possibilidade de segunda instância` pode cair por terra gerando problemas que ninguém vai saber porque estão ocorrendo;

- quando `thread safe` e prematura, realiza uma carga prematura (talvez desnessária) de um objeto que pode jamais ser utilizado durante a execução da aplicação. Isso pode sim gerar desperdício de memória. Mas, aqui temos também uma oportunidade de colocar no `startup` da aplicação a carga de algum recurso que sabemos que vamos utilizar e que é custoso de levantar em memória;

Na prática, o uso depende não apenas na regra de negócio, mas também das decisões arquiteturais que você toma na elaboração do código.