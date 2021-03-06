## Ciclos de Referências Podem Vazar Memória

As garantias de segurança de memória do Rust tornam _difícil_ mas não impossível
acidentalmente criar memória que nunca é liberada (conhecida como um _vazamento
de memória_, ou _memory leak_). O Rust garante em tempo de compilação que não
haverá corridas de dados, mas não garante a prevenção de vazamentos de memória
por completo da mesma forma, o que significa que vazamentos de memória são
_memory safe_ em Rust. Podemos ver que o Rust permite vazamentos de memória
usando `Rc<T>` e `RefCell<T>`: é possível criar referências onde os itens se
referem uns aos outros em um ciclo. Isso cria vazamentos de memória porque a
contagem de referências de cada item no ciclo nunca chegará a 0, e os valores
nunca serão destruídos.

### Criando um Ciclo de Referências

Vamos dar uma olhada em como um ciclo de referências poderia acontecer e como
preveni-lo, começando com a definição do enum `List` e um método `tail`
(_cauda_) na Listagem 15-25:

<span class="filename">Arquivo: src/main.rs</span>

<!-- A fn main escondida (primeira linha) está aqui para desabilitar o wrapping
automático em uma fn main que os doc tests fazem; o `use List` falha se esta
listagem é colocada dentro de uma main -->

```rust
# fn main() {}
use std::rc::Rc;
use std::cell::RefCell;
use List::{Cons, Nil};

#[derive(Debug)]
enum List {
    Cons(i32, RefCell<Rc<List>>),
    Nil,
}

impl List {
    fn tail(&self) -> Option<&RefCell<Rc<List>>> {
        match *self {
            Cons(_, ref item) => Some(item),
            Nil => None,
        }
    }
}
```

<span class="caption">Listagem 15-25: Uma definição de cons list que contém um
`RefCell<T>` para que possamos modificar ao que se refere uma variante
`Cons`</span>

Estamos usando outra variação da definição de `List` da Listagem 15-5. O segundo
elemento na variante `Cons` agora é um `RefCell<Rc<List>>`, o que significa que
em vez de ter a habilidade de modificar o valor `i32` como fizemos na Listagem
15-24, queremos modificar a `List` à qual a variante `Cons` está apontando.
Também estamos adicionando um método `tail` para nos facilitar o acesso ao
segundo item quando tivermos uma variante `Cons`.

Na Listagem 15-26, estamos adicionando uma função `main` que usa as definições
da Listagem 15-25. Este código cria uma lista em `a` e uma lista em `b` que
aponta para a lista em `a`, e depois modifica a lista em `a` para apontar para
`b`, o que cria um ciclo de referências. Temos declarações de `println!` ao
longo do caminho para mostrar quais são as contagens de referências em vários
pontos do processo:

<span class="filename">Arquivo: src/main.rs</span>

```rust
# use List::{Cons, Nil};
# use std::rc::Rc;
# use std::cell::RefCell;
# #[derive(Debug)]
# enum List {
#     Cons(i32, RefCell<Rc<List>>),
#     Nil,
# }
#
# impl List {
#     fn tail(&self) -> Option<&RefCell<Rc<List>>> {
#         match *self {
#             Cons(_, ref item) => Some(item),
#             Nil => None,
#         }
#     }
# }
#
fn main() {
    let a = Rc::new(Cons(5, RefCell::new(Rc::new(Nil))));

    println!("a: contagem de referências inicial = {}", Rc::strong_count(&a));
    println!("a: próximo item = {:?}", a.tail());

    let b = Rc::new(Cons(10, RefCell::new(Rc::clone(&a))));

    println!("a: contagem de referências depois da criação de b = {}", Rc::strong_count(&a));
    println!("b: contagem de referências inicial = {}", Rc::strong_count(&b));
    println!("b: próximo item = {:?}", b.tail());

    if let Some(link) = a.tail() {
        *link.borrow_mut() = Rc::clone(&b);
    }

    println!("b: contagem de referências depois de mudar a = {}", Rc::strong_count(&b));
    println!("a: contagem de referências depois de mudar a = {}", Rc::strong_count(&a));

    // Descomente a próxima linha para ver que temos um ciclo; ela irá
    // estourar a pilha
    // println!("a: próximo item = {:?}", a.tail());
}
```

<span class="caption">Listagem 15-26: Criando um ciclo de referências de dois
valores `List` apontando um para o outro</span>

Nós criamos uma instância de `Rc<List>` segurando um valor `List` na variável
`a` com a lista inicial de `5, Nil`. Então criamos uma instância de `Rc<List>`
segurando outro valor `List` na variável `b` que contém o valor 10 e aponta para
a lista em `a`.

Nós modificamos `a` para que aponte para `b` em vez de `Nil`, o que cria um
ciclo. Fazemos isso usando o método `tail` para obter uma referência ao
`RefCell<Rc<List>>` em `a`, a qual colocamos na variável `link`. Então usamos o
método `borrow_mut` no `RefCell<Rc<List>>` para modificar o valor interno: de um
`Rc<List>` que guarda um valor `Nil` para o `Rc<List>` em `b`.

Quando rodamos esse código, mantendo o último `println!` comentado por ora,
obtemos esta saída:

```text
a: contagem de referências inicial = 1
a: próximo item = Some(RefCell { value: Nil })
a: contagem de referências depois da criação de b = 2
b: contagem de referências inicial = 1
b: próximo item = Some(RefCell { value: Cons(5, RefCell { value: Nil }) })
b: contagem de referências depois de mudar a = 2
a: contagem de referências depois de mudar a = 2
```

A contagem de referências das instâncias de `Rc<List>` em ambos `a` e `b` é 2
depois que mudamos a lista em `a` para apontar para `b`. No final da `main`, o
Rust tentará destruir `b` primeiro, o que diminuirá em 1 a contagem em cada uma
das instâncias de `Rc<List>` em `a` e `b`.

Contudo, como a variável `a` ainda está se referindo ao `Rc<List>` que estava em
`b`, ele terá uma contagem de 1 em vez de 0, então a memória que ele tem no heap
não será destruída. A memória irá ficar lá com uma contagem de 1, para sempre.
Para visualizar esse ciclo de referências, criamos um diagrama na Figura 15-4:

<img alt="Ciclo de referências de listas" src="img/trpl15-04.svg" class="center"
/>

<span class="caption">Figura 15-4: Um ciclo de referências das listas `a` e `b`
apontando uma para a outra</span>

Se você descomentar o último `println!` e rodar o programa, o Rust tentará
imprimir esse ciclo com `a` apontando para `b` apontando para `a` e assim por
diante até estourar a pilha.

Nesse exemplo, logo depois que criamos o ciclo de referências, o programa
termina. As consequências desse ciclo não são muito graves. Se um programa mais
complexo aloca um monte de memória em um ciclo e não a libera por muito tempo,
ele acaba usando mais memória do que precisa e pode sobrecarregar o sistema,
fazendo com que fique sem memória disponível.

Criar ciclos de referências não é fácil de fazer, mas também não é impossível.
Se você tem valores `RefCell<T>` que contêm valores `Rc<T>` ou combinações
aninhadas de tipos parecidas, com mutabilidade interior e contagem de
referências, você deve se assegurar de que não está criando ciclos; você não
pode contar com o Rust para pegá-los. Criar ciclos de referências seria um erro
de lógica no seu programa, e você deve usar testes automatizados, revisões de
código e outras práticas de desenvolvimento de software para minimizá-los.

Outra solução para evitar ciclos de referências é reorganizar suas estruturas de
dados para que algumas referências expressem posse e outras não. Assim, você
pode ter ciclos feitos de algumas relações de posse e algumas relações de não
posse, e apenas as relações de posse afetam se um valor pode ou não ser
destruído. Na Listagem 15-25, nós sempre queremos que as variantes `Cons`
possuam sua lista, então reorganizar a estrutura de dados não é possível. Vamos
dar uma olhada em um exemplo usando grafos feitos de vértices pais e vértices
filhos para ver quando relações de não posse são um jeito apropriado de evitar
ciclos de referências.

### Prevenindo Ciclos de Referência: Transforme um `Rc<T>` em um `Weak<T>`

Até agora, demonstramos que chamar `Rc::clone` aumenta a `strong_count`
(_contagem de referências fortes_) de uma instância `Rc<T>`, e que a instância
`Rc<T>` só é liberada se sua `strong_count` é 0. Também podemos criar uma
_referência fraca_ (_weak reference_) ao valor dentro de uma instância `Rc<T>`
chamando `Rc::downgrade` e passando-lhe uma referência ao `Rc<T>`. Quando
chamamos `Rc::downgrade`, nós obtemos um ponteiro inteligente do tipo `Weak<T>`.
Em vez de aumentar em 1 a `strong_count` na instância `Rc<T>`, chamar
`Rc::downgrade` aumenta em 1 a `weeak_count` (_contagem de referências fracas_).
O tipo `Rc<T>` usa a `weak_count` para registrar quantas referências `Weak<T>`
existem, parecido com a `strong_count`. A diferença é que a `weak_count` não
precisa ser 0 para a instância `Rc<T>` ser destruída.

Referências fortes são o modo como podemos compartilhar posse de uma instância
`Rc<T>`. Referências fracas não expressam uma relação de posse. Elas não irão
causar um ciclo de referências porque qualquer ciclo envolvendo algumas
referências fracas será quebrado uma vez que a contagem de referências fortes
dos valores envolvidos for 0.

Como o valor ao qual o `Weak<T>` faz referência pode ter sido destruído, para
fazer qualquer coisa com ele, precisamos nos assegurar de que ele ainda exista.
Fazemos isso chamando o método `upgrade` na instância `Weak<T>`, o que nos
retornará uma `Option<Rc<T>>`. Iremos obter um resultado de `Some` se o valor do
`Rc<T>` ainda não tiver sido destruído e um resultado de `None` caso ele já
tenha sido destruído. Como o `upgrade` retorna uma `Option<T>`, o Rust irá
garantir que lidemos com ambos os casos `Some` e `None`, e não haverá um
ponteiro inválido.

Como exemplo, em vez de usarmos uma lista cujos itens sabem apenas a respeito do
próximo item, iremos criar uma árvore cujos itens sabem sobre seus itens filhos
_e_ sobre seus itens pais.

#### Criando uma Estrutura de Dados em Árvore: Um `Vertice` com Vértices Filhos

Para começar, vamos construir uma árvore com vértices que saibam apenas sobre
seus vértices filhos. Iremos criar uma estrutura chamada `Vertice` que contenha
seu próprio valor `i32`, além de referências para seus valores filhos do tipo
`Vertice`:

<span class="filename">Arquivo: src/main.rs</span>

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
struct Vertice {
    valor: i32,
    filhos: RefCell<Vec<Rc<Vertice>>>,
}
```

Queremos que um `Vertice` tenha posse de seus filhos, e queremos compartilhar
essa posse com variáveis para que possamos acessar cada `Vertice` da árvore
diretamente. Para fazer isso, definimos os itens do `Vec<T>` para serem valores
do tipo `Rc<Vertice>`. Também queremos modificar quais vértices são filhos de
outro vértice, então temos um `RefCell<T>` em `filhos` em volta do
`Vec<Rc<Vertice>>`.

Em seguida, iremos usar nossa definição de struct e criar uma instância de
`Vertice` chamada `folha` com o valor 3 e nenhum filho, e outra instância
chamada `galho` com o valor 5 e `folha` como um de seus filhos, como mostra a
Listagem 15-27:

<span class="filename">Arquivo: src/main.rs</span>

```rust
# use std::rc::Rc;
# use std::cell::RefCell;
#
# #[derive(Debug)]
# struct Vertice {
#     valor: i32,
#     filhos: RefCell<Vec<Rc<Vertice>>>,
# }
#
fn main() {
    let folha = Rc::new(Vertice {
        valor: 3,
        filhos: RefCell::new(vec![]),
    });

    let galho = Rc::new(Vertice {
        valor: 5,
        filhos: RefCell::new(vec![Rc::clone(&folha)]),
    });
}
```

<span class="caption">Listagem 15-27: Criando um vértice `folha` sem filhos e um
vértice `galho` com `folha` como um de seus filhos</span>

Nós clonamos o `Rc<Vertice>` em `folha` e armazenamos o resultado em `galho`, o
que significa que o `Vertice` em `folha` agora tem dois possuidores: `folha` e
`galho`. Podemos ir de `galho` para `folha` através de `galho.filhos`, mas não
temos como ir de `folha` para `galho`. O motivo é que `folha` não tem referência
a `galho` e não sabe que eles estão relacionados. Queremos que `folha` saiba que
`galho` é seu pai. Faremos isso em seguida.

#### Adicionando uma Referência de um Filho para o Seu Pai

Para tornar o vértice filho ciente de seu pai, precisamos adicionar um campo
`pai` a nossa definição da struct `Vertice`. O problema é decidir qual deveria
ser o tipo de `pai`. Sabemos que ele não pode conter um `Rc<T>` porque isso
criaria um ciclo de referências com `folha.pai` apontando para `galho` e
`galho.filhos` apontando para `folha`, o que faria com que seus valores de
`strong_count` nunca chegassem a 0.

Pensando sobre as relações de outra forma, um vértice pai deveria ter posse de
seus filhos: se um vértice pai é destruído, seus vértices filhos também deveriam
ser. Entretanto, um filho não deveria ter posse de seu pai: se destruirmos um
vértice filho, o pai ainda deveria existir. Esse é um caso para referências
fracas!

Então em vez de `Rc<T>`, faremos com que o tipo de `pai` use `Weak<T>`, mais
especificamente um `RefCell<Weak<Vertice>>`. Agora nossa definição da struct
`Vertice` fica assim:

<span class="filename">Arquivo: src/main.rs</span>

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[derive(Debug)]
struct Vertice {
    valor: i32,
    pai: RefCell<Weak<Vertice>>,
    filhos: RefCell<Vec<Rc<Vertice>>>,
}
```

Agora um vértice pode se referir a seu vértice pai, mas não tem posse dele. Na
Listagem 15-28, atualizamos a `main` com essa nova definição para que o vértice
`folha` tenha um jeito de se referir a seu pai, `galho`:

<span class="filename">Arquivo: src/main.rs</span>

```rust
# use std::rc::{Rc, Weak};
# use std::cell::RefCell;
#
# #[derive(Debug)]
# struct Vertice {
#     valor: i32,
#     pai: RefCell<Weak<Vertice>>,
#     filhos: RefCell<Vec<Rc<Vertice>>>,
# }
#
fn main() {
    let folha = Rc::new(Vertice {
        valor: 3,
        pai: RefCell::new(Weak::new()),
        filhos: RefCell::new(vec![]),
    });

    println!("pai de folha = {:?}", folha.pai.borrow().upgrade());

    let galho = Rc::new(Vertice {
        valor: 5,
        pai: RefCell::new(Weak::new()),
        filhos: RefCell::new(vec![Rc::clone(&folha)]),
    });

    *folha.pai.borrow_mut() = Rc::downgrade(&galho);

    println!("pai de folha = {:?}", folha.pai.borrow().upgrade());
}
```

<span class="caption">Listagem 15-28: Um vértice `folha` com uma referência
`Weak` a seu vértice pai `galho`</span>

Criar o vértice `folha` é semelhante a como o criamos na Listagem 15-27, com
exceção do campo `pai`: `folha` começa sem um pai, então criamos uma instância
nova e vazia de uma referência `Weak<Vertice>`.

Nesse ponto, quando tentamos obter uma referência ao pai de `folha` usando o
método `upgrade`, recebemos um valor `None`. Vemos isso na saída do primeiro
comando `println!`:

```text
pai de folha = None
```

Quando criamos o vértice `galho`, ele também tem uma nova referência
`Weak<Vertice>` no campo `pai`, porque `galho` não tem um vértice pai. Nós ainda
temos `folha` como um dos filhos de `galho`. Uma vez que temos a instância de
`Vertice` em `galho`, podemos modificar `folha` para lhe dar uma referência
`Weak<Vertice>` a seu pai. Usamos o método `borrow_mut` do
`RefCell<Weak<Vertice>>` no campo `pai` de `folha`, e então usamos a função
`Rc::downgrade` para criar uma referência `Weak<Vertice>` a `galho` a partir do
`Rc<Vertice>` em `galho`.

Quando imprimimos o pai de `folha` de novo, dessa vez recebemos uma variante
`Some` contendo `galho`: agora `folha` tem acesso a seu pai! Quando imprimimos
`folha`, nós também evitamos o ciclo que eventualmente terminou em um estouro de
pilha como o que tivemos na Listagem 15-26: as referências `Weak<Vertice>` são
impressas como `(Weak)`:

```text
pai de folha = Some(Vertice { valor: 5, pai: RefCell { valor: (Weak) },
filhos: RefCell { valor: [Vertice { valor: 3, pai: RefCell { valor: (Weak) },
filhos: RefCell { valor: [] } }] } })
```

A falta de saída infinita indica que esse código não criou um ciclo de
referências. Também podemos perceber isso olhando para os valores que obtemos ao
chamar `Rc::strong_count` e `Rc::weak_count`.

#### Visualizando Mudanças a `strong_count` e `weak_count`

Para ver como os valores de `strong_count` e `weak_count` das instâncias de
`Rc<Vertice>` mudam, vamos criar um novo escopo interno e mover a criação de
`galho` para dentro dele. Fazendo isso, podemos ver o que acontece quando
`galho` é criado e depois destruído quando sai de escopo. As modificações são
mostradas na Listagem 15-29:

<span class="filename">Arquivo: src/main.rs</span>

```rust
# use std::rc::{Rc, Weak};
# use std::cell::RefCell;
#
# #[derive(Debug)]
# struct Vertice {
#     valor: i32,
#     pai: RefCell<Weak<Vertice>>,
#     filhos: RefCell<Vec<Rc<Vertice>>>,
# }
#
fn main() {
    let folha = Rc::new(Vertice {
        valor: 3,
        pai: RefCell::new(Weak::new()),
        filhos: RefCell::new(vec![]),
    });

    println!(
        "folha: fortes = {}, fracas = {}",
        Rc::strong_count(&folha),
        Rc::weak_count(&folha),
    );

    {
        let galho = Rc::new(Vertice {
            valor: 5,
            pai: RefCell::new(Weak::new()),
            filhos: RefCell::new(vec![Rc::clone(&folha)]),
        });

        *folha.pai.borrow_mut() = Rc::downgrade(&galho);

        println!(
            "galho: fortes = {}, fracas = {}",
            Rc::strong_count(&galho),
            Rc::weak_count(&galho),
        );

        println!(
            "folha: fortes = {}, fracas = {}",
            Rc::strong_count(&folha),
            Rc::weak_count(&folha),
        );
    }

    println!("pai de folha = {:?}", folha.pai.borrow().upgrade());
    println!(
        "folha: fortes = {}, fracas = {}",
        Rc::strong_count(&folha),
        Rc::weak_count(&folha),
    );
}
```

<span class="caption">Listagem 15-29: Criando `galho` em um escopo interno e
examinando contagens de referências fortes e fracas</span>

Depois que `folha` é criada, seu `Rc<Vertice>` tem uma _strong count_ de 1 e uma
_weak count_ de 0. Dentro do escopo interno, criamos `galho` e o associamos a
`folha`. Nesse ponto, quando imprimimos as contagens, o `Rc<Vertice>` em `galho`
tem uma strong count de 1 e uma weak count de 1 (porque `folha.pai` aponta para
`galho` com uma `Weak<Vertice>`). Quando imprimirmos as contagens de `folha`,
veremos que ela terá uma strong count de 2, porque `galho` agora tem um clone do
`Rc<Vertice>` de `folha` armazenado em `galho.filhos`, mas ainda terá uma weak
count de 0.

Quando o escopo interno termina, `galho` sai de escopo e a strong count do
`Rc<Vertice>` diminui para 0, e então seu `Vertice` é destruído. A weak count de
1 por causa de `folha.pai` não tem nenhuma influência sobre se `Vertice` é
destruído ou não, então não temos nenhum vazamento de memória!

Se tentarmos acessar o pai de `folha` depois do fim do escopo, receberemos
`None` de novo. No fim do programa, o `Rc<Vertice>` em `folha` tem uma strong
count de 1 e uma weak count de 0, porque a variável `folha` agora é de novo a
única referência ao `Rc<Vertice>`.

Toda a lógica que gerencia as contagens e a destruição de valores faz parte de
`Rc<T>` e `Weak<T>` e suas implementações da trait `Drop`. Ao especificarmos na
definição de `Vertice` que a relação de um filho para o seu pai deva ser uma
referência `Weak<T>`, somos capazes de ter vértices pai apontando para para
vértices filho e vice-versa sem criar ciclos de referência e vazamentos de
memória.

## Resumo

Esse capítulo cobriu como usar ponteiros inteligentes para fazer garantias e
trade-offs diferentes daqueles que o Rust faz por padrão com referências
normais. O tipo `Box<T>` tem um tamanho conhecido e aponta para dados alocados
no heap. O tipo `Rc<T>` mantém registro do número de referências a dados no
heap, para que eles possam ter múltiplos possuidores. O tipo `RefCell<T>` com
sua mutabilidade interior nos dá um tipo que podemos usar quando precisamos de
um tipo imutável mas precisamos mudar um valor interno ao tipo; ele também
aplica as regras de empréstimo em tempo de execução em vez de em tempo de
compilação.

Também foram discutidas as traits `Deref` e `Drop` que tornam possível muito da
funcionalidade dos ponteiros inteligentes. Exploramos ciclos de referências que
podem causar vazamentos de memória e como preveni-los usando `Weak<T>`.

Se esse capítulo tiver aguçado seu interesse e você quiser implementar seus
próprios ponteiros inteligentes, dê uma olhada no "Rustnomicon" em
_https://doc.rust-lang.org/stable/nomicon/_ para mais informação útil.

Em seguida, conversaremos sobre concorrência em Rust. Você irá até aprender
sobre alguns novos ponteiros inteligentes.
