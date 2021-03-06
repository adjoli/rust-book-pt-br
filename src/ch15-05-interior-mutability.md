## `RefCell<T>` e a Pattern de Mutabilidade Interior

_Mutabilidade interior_ (_interior mutability_) é uma design pattern em Rust que
lhe permite modificar um dado mesmo quando há referências imutáveis a ele:
normalmente, esta ação é proibida pelas regras de empréstimo. Para fazer isso, a
pattern usa código `unsafe` (_inseguro_) dentro de uma estrutura de dados para
dobrar as regras normais do Rust que governam mutação e empréstimo. Nós ainda
não cobrimos código unsafe; faremos isso no Capítulo 19. Podemos usar tipos que
usam a pattern de mutabilidade interior quando podemos garantir que as regras de
empréstimo serão seguidas em tempo de execução, ainda que o compilador não o
possa garantir. O código `unsafe` envolvido é então embrulhado em uma API safe,
e o tipo exterior permanece imutável.

Para explorar este conceito, vamos ver o tipo `RefCell<T>` que segue a pattern
de mutabilidade interior.

### Aplicando Regras de Empréstimo em Tempo de Execução com o `RefCell<T>`

Diferente do `Rc<T>`, o tipo `RefCell<T>` representa posse única sobre o dado
que ele contém. Então o que torna o `RefCell<T>` diferente de um tipo como o
`Box<T>`? Lembre-se das regras de empréstimo que você aprendeu no Capítulo 4:

- Em qualquer momento, você pode ter _um dos_ mas não ambos os seguintes: uma
  única referência mutável _ou_ qualquer número de referências imutáveis;
- Referências devem sempre ser válidas.

Com referências e com o `Box<T>`, as invariantes das regras de empréstimo são
aplicadas em tempo de compilação. Com o `RefCell<T>`, essas invariantes são
aplicadas _em tempo de execução_. Com referências, se você quebra essas regras,
você recebe um erro de compilação. Com o `RefCell<T>`, se você quebrar essas
regras, seu programa irá sofrer um `panic!` e terminar.

As vantagens de checar as regras de empréstimo em tempo de compilação são que
erros são pegos mais cedo no processo de desenvolvimento, e não há nenhum custo
de desempenho de execução porque toda a análise é completada de antemão. Por
esses motivos, checar as regras de empréstimo em tempo de compilação é a melhor
opção na maioria dos casos, e por isso este é o padrão do Rust.

A vantagem de checar as regras de empréstimo em tempo de execução,
alternativamente, é que certos cenários _memory-safe_ (_seguros em termos de
memória_) são então permitidos, ao passo que seriam proibidos pelas checagens em
tempo de compilação. A análise estática, como a do compilador Rust, é
inerentemente conservadora. Algumas propriedades do programa são impossíveis de
detectar analisando o código: o exemplo mais famoso é o Problema da Parada, que
está além do escopo deste livro mas é um tópico interessante para pesquisa.

Como algumas análises são impossíveis, se o compilador Rust não consegue se
assegurar que o código obedece às regras de posse, ele pode rejeitar um programa
correto; neste sentido, ele é conservador. Se o Rust aceitasse um programa
incorreto, os usuários não poderiam confiar nas garantias que ele faz. Se, por
outro lado, o Rust rejeita um programa correto, o programador terá alguma
inconveniência, mas nada catastrófico pode acontecer. O tipo `RefCell<T>` é útil
quando você tem certeza que seu código segue as regras de empréstimo, mas o
compilador é incapaz de entender e garantir isso.

Assim como o `Rc<T>`, o `RefCell<T>` é apenas para uso em cenários de thread
única e lhe darão um erro de compilação se você tentar usá-lo em um contexto de
múltiplas threads. Falaremos sobre como obter a funcionalidade de um
`RefCell<T>` em um programa multithread no Capítulo 16.

Aqui está uma recapitulação das razões para escolher o `Box<T>`, o `Rc<T>` ou o
`RefCell<T>`:

- O `Rc<T>` permite múltiplos possuidores do mesmo dado; `Box<T>` e `RefCell<T>`
  têm possuidores únicos.
- O `Box<T>` permite empréstimos imutáveis ou mutáveis checados em tempo de
  compilação; o `Rc<T>` permite apenas empréstimos imutáveis em tempo de
  compilação; o `RefCell<T>` permite empréstimos imutáveis ou mutáveis checados
  em tempo de execução.
- Como o `RefCell<T>` permite empréstimos mutáveis checados em tempo de
  execução, nós podemos modificar o valor dentro de um `RefCell<T>` mesmo quando
  o `RefCell<T>` é imutável.

Modificar o valor dentro de um valor imutável é a pattern de _mutabilidade
interior_. Vamos dar uma olhada em uma situação em que a mutabilidade interior é
útil e examinar como ela é possível.

### Mutabilidade Interior: Um Empréstimo Mutável de um Valor Imutável

Uma consequência das regras de empréstimo é que quando temos um valor imutável,
nós não podemos pegá-lo emprestado mutavelmente. Por exemplo, este código não
compila:

```rust,ignore
fn main() {
    let x = 5;
    let y = &mut x;
}
```

Quando tentamos compilar este código, recebemos o seguinte erro:

```text
erro[E0596]: não posso pegar emprestado a variável local imutável `x` como
mutável
 --> src/main.rs:3:18
  |
2 |     let x = 5;
  |         - considere mudar isto para `mut x`
3 |     let y = &mut x;
  |                  ^ não posso pegar emprestado mutavelmente
```

Contudo, há situações em que seria útil para um valor modificar a si mesmo em
seus métodos, mas continuar parecendo imutável para código externo. Código fora
dos métodos do valor não teriam como modificá-lo. Usar o `RefCell<T>` é um jeito
de obter a habilidade de ter mutabilidade interior. Mas o `RefCell<T>` não dá a
volta nas regras de empréstimo por completo: o borrow checker no compilador
permite esta mutabilidade interior, e as regras de empréstimo são em vez disso
checadas em tempo de execução. Se violarmos as regras, receberemos um `panic!`
em vez de um erro de compilação.

Vamos trabalhar com um exemplo prático onde podemos usar o `RefCell<T>` para
modificar um valor imutável e ver por que isto é útil.

#### Um Caso de Uso para a Mutabilidade Interior: Objetos Simulados

Um _dublê de teste_ (_test double_) é um conceito geral de programação para um
tipo usado no lugar de outro durante os testes. _Objetos simulados_ (_mock
objects_) são tipos específicos de dublês de teste que registram o que acontece
durante o teste para que possamos confirmar que as ações corretas aconteceram.

Rust não tem objetos da mesma forma que outras linguagens, e não tem
funcionalidade de objetos simulados embutida na biblioteca padrão como algumas
outras linguagens. Contudo, certamente podemos criar uma struct que serve os
mesmos propósitos que um objeto simulado.

Eis o cenário que vamos testar: vamos criar uma biblioteca que acompanha um
valor contra um valor máximo e envia mensagens com base em quão próximo do valor
máximo o valor atual está. Esta biblioteca pode ser usada para acompanhar a cota
de um usuário para o número de chamadas de API que ele tem direito a fazer, por
exemplo.

Nossa biblioteca irá prover somente a funcionalidade de acompanhar quão perto do
máximo um valor está e o que as mensagens deveriam ser em quais momentos. As
aplicações que usarem nossa biblioteca terão a responsabilidade de prover o
mecanismo para enviar as mensagens: a aplicação pode pôr a mensagem na própria
aplicação, enviar um email, uma mensagem de texto, ou alguma outra coisa. A
biblioteca não precisa saber deste detalhe. Tudo que ela precisa é de algo que
implemente uma trait que iremos prover chamada `Mensageiro`. A
Listagem 15-20 mostra o código da biblioteca:

<span class="filename">Arquivo: src/lib.rs</span>

```rust
pub trait Mensageiro {
    fn enviar(&self, msg: &str);
}

pub struct AvisaLimite<'a, T: 'a + Mensageiro> {
    mensageiro: &'a T,
    valor: usize,
    max: usize,
}

impl<'a, T> AvisaLimite<'a, T>
    where T: Mensageiro {
    pub fn new(mensageiro: &T, max: usize) -> AvisaLimite<T> {
        AvisaLimite {
            mensageiro,
            valor: 0,
            max,
        }
    }

    pub fn set_valor(&mut self, valor: usize) {
        self.valor = valor;

        let porcentagem_do_max = self.valor as f64 / self.max as f64;

        if porcentagem_do_max >= 0.75 && porcentagem_do_max < 0.9 {
            self.mensageiro.enviar("Aviso: Você usou mais de 75% da sua cota!");
        } else if porcentagem_do_max >= 0.9 && porcentagem_do_max < 1.0 {
            self.mensageiro.enviar("Aviso urgente: Você usou mais de 90% da sua cota!");
        } else if porcentagem_do_max >= 1.0 {
            self.mensageiro.enviar("Erro: Você excedeu sua cota!");
        }
    }
}
```

<span class="caption">Listagem 15-20: Uma biblioteca para acompanhar quão perto
do máximo um valor está e avisar quando o valor está em certos níveis</span>

Uma parte importante deste código é que a trait `Mensageiro` tem um método
chamado `enviar` que recebe uma referência imutável a `self` e o texto da
mensagem. Esta é a interface que nosso objeto simulado precisa ter. A outra
parte importante é que queremos testar o comportamento do método `set_valor` no
`AvisaLimite`. Podemos mudar o que passamos para o parâmetro `valor`, mas o
`set_valor` não retorna nada sobre o qual possamos fazer asserções. Queremos
poder dizer que se criarmos um `AvisaLimite` com algo que implemente a trait
`Mensageiro` e um valor específico de `max`, quando passarmos diferentes números
para o `valor`, o mensageiro receberá o comando para enviar as mensagens
apropriadas.

Precisamos de um objeto simulado que, em vez de enviar um email ou mensagem de
texto quando chamarmos `enviar`, irá apenas registrar as mensagens que recebeu
para enviar. Podemos criar uma nova instância do objeto simulado, criar um
`AvisaLimite` que use o objeto simulado, chamar o método `set_valor` no
`AvisaLimite`, e então verificar se o objeto simulado tem as mensagens que
esperamos. A Listagem 15-21 mostra uma tentativa de implementar um objeto
simulado para fazer exatamente isto, mas que o borrow checker não permite:

<span class="filename">Arquivo: src/lib.rs</span>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    struct MensageiroSimulado {
        mensagens_enviadas: Vec<String>,
    }

    impl MensageiroSimulado {
        fn new() -> MensageiroSimulado {
            MensageiroSimulado { mensagens_enviadas: vec![] }
        }
    }

    impl Mensageiro for MensageiroSimulado {
        fn enviar(&self, mensagem: &str) {
            self.mensagens_enviadas.push(String::from(mensagem));
        }
    }

    #[test]
    fn envia_uma_mensagem_de_aviso_de_acima_de_75_porcento() {
        let mensageiro_simulado = MensageiroSimulado::new();
        let mut avisa_limite = AvisaLimite::new(&mensageiro_simulado, 100);

        avisa_limite.set_valor(80);

        assert_eq!(mensageiro_simulado.mensagens_enviadas.len(), 1);
    }
}
```

<span class="caption">Listagem 15-21: Uma tentativa de implementar um
`MensageiroSimulado` que não é permitida pelo borrow checker</span>

Este código de teste define uma struct `MensageiroSimulado` que tem um campo
`mensagens_enviadas` com um `Vec` de valores `String` para registrar as
mensagens que ele recebe para enviar. Também definimos uma função associada
`new` para facilitar a criação de novos valores `MensageiroSimulado` que começam
com uma lista vazia de mensagens. Então implementamos a trait `Mensageiro` para
o `MensageiroSimulado` para que possamos passar um `MensageiroSimulado` a um
`AvisaLimite`. Na definição do método `enviar`, nós pegamos a mensagem passada
como parâmetro e a armazenamos na lista `mensagens_enviadas` do
`MensageiroSimulado`.

No teste, estamos testando o que acontece quando o `AvisaLimite` recebe o
comando para setar o `valor` para algo que é mais do que 75 porcento do valor
`max`. Primeiro, criamos um novo `MensageiroSimulado`, que irá começar com uma
lista vazia de mensagens. Então criamos um novo `AvisaLimite` e lhe damos uma
referência ao novo `MensageiroSimulado` e um valor `max` de 100. Nós chamamos o
método `set_valor` no `AvisaLimite` com um valor de 80, que é mais do que 75
porcento de 100. Então conferimos se a lista de mensagens que o
`MensageiroSimulado` está registrando agora tem uma mensagem nela.

Entretanto, há um problema neste teste, conforme abaixo:

```text
erro[E0596]: não posso pegar emprestado o campo imutável
             `self.mensagens_enviadas` como mutável
  --> src/lib.rs:52:13
   |
51 |         fn send(&self, message: &str) {
   |                 ----- use `&mut self` aqui para torná-lo mutável
52 |             self.sent_messages.push(String::from(message));
   |             ^^^^^^^^^^^^^^^^^^ não posso pegar emprestado mutavelmente um
                                    campo imutável
```

Não podemos modificar o `MensageiroSimulado` para registrar as mensagens porque
o método `enviar` recebe uma referência imutável a `self`. Também não podemos
seguir a sugestão do texto de erro e usar `&mut self` em vez disso porque a
assinatura de `enviar` não corresponderia à assinatura na definição da trait
`Mensageiro` (fique à vontade para tentar e ver qual mensagem de erro você
recebe).

Esta é uma situação em que a mutabilidade interior pode ajudar! Vamos armazenas
as `mensagens_enviadas` dentro de um `RefCell<T>`, e então o método `enviar`
poderá modificar `mensagens_enviadas` para armazenar as mensagens que já vimos.
A Listagem 15-22 mostra como fica isto:

<span class="filename">Arquivo: src/lib.rs</span>

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    struct MensageiroSimulado {
        mensagens_enviadas: RefCell<Vec<String>>,
    }

    impl MensageiroSimulado {
        fn new() -> MensageiroSimulado {
            MensageiroSimulado { mensagens_enviadas: RefCell::new(vec![]) }
        }
    }

    impl Mensageiro for MensageiroSimulado {
        fn enviar(&self, mensagem: &str) {
            self.mensagens_enviadas.borrow_mut().push(String::from(mensagem));
        }
    }

    #[test]
    fn envia_uma_mensagem_de_aviso_de_acima_de_75_porcento() {
        // --snip--
#         let mensageiro_simulado = MensageiroSimulado::new();
#         let mut avisa_limite = AvisaLimite::new(&mensageiro_simulado, 100);
#         avisa_limite.set_valor(75);

        assert_eq!(mensageiro_simulado.mensagens_enviadas.borrow().len(), 1);
    }
}
```

<span class="caption">Listagem 15-22: Usando `RefCell<T>` para modificar um
valor interno enquanto o valor externo é considerado imutável</span>

O campo `mensagens_enviadas` agora é do tipo `RefCell<Vec<String>>` em vez de
`Vec<String>`. Na função `new`, nós criamos uma nova instância de
`RefCell<Vec<String>>` em torno do vetor vazio.

Para a implementação do método `enviar`, o primeiro parâmetro ainda é um
empréstimo imutável de `self`, que corresponde à definição da trait. Nós
chamamos `borrow_mut` no `RefCell<Vec<String>>` em `self.mensagens_enviadas`
para obter uma referência mutável ao valor dentro do `RefCell<Vec<String>>`, que
é o vetor. Então podemos chamar `push` na referência mutável ao vetor para
registrar as mensagens enviadas durante o teste.

A última mudança que temos que fazer é na asserção: para ver quantos itens estão
no vetor interno, chamamos `borrow` no `RefCell<Vec<String>>` para obter uma
referência imutável ao vetor.

Agora que você viu como usar o `RefCell<T>`, vamos nos aprofundar em como ele
funciona!

#### O `RefCell<T>` Registra Empréstimos em Tempo de Execução

Quando estamos criando referências imutáveis e mutáveis, usamos as sintaxes `&`
e `&mut`, respectivamente. Com o `RefCell<T>`, usamos os métodos `borrow` e
`borrow_mut`, que são parte da API safe que pertence ao `RefCell<T>`. O método
`borrow` retorna o ponteiro inteligente `Ref<T>`, e o `borrow_mut` retorna o
ponteiro inteligente `RefMut<T>`. Ambos os tipos implementam `Deref`, então
podemos tratá-los como referências normais.

O tipo `RefCell<T>` mantém registro de quantos ponteiros inteligentes `Ref<T>` e
`RefMut<T>` estão atualmente ativos. Cada vez que chamamos `borrow`, o
`RefCell<T>` aumenta seu contador de quantos empréstimos imutáveis estão ativos.
Quando um valor `Ref<T>` sai de escopo, o contador de empréstimos imutáveis
diminui em um. Assim como as regras de empréstimo em tempo de compilação, o
`RefCell<T>` nos permite ter vários empréstimos imutáveis ou um empréstimo
mutável em um dado momento.

Se tentarmos violar estas regras, em vez de receber um erro do compilador como
iríamos com referências, a implementação de `RefCell<T>` chamará `panic!` em
tempo de execução. A Listagem 15-23 mostra uma modificação da implementação do
`enviar` da Listagem 15-22. Estamos deliberadamente tentando criar dois
empréstimos mutáveis ativos para o mesmo escopo para ilustrar que o `RefCell<T>`
nos impede de fazer isto em tempo de execução:

<span class="filename">Arquivo: src/lib.rs</span>

```rust,ignore
impl Mensageiro for MensageiroSimulado {
    fn enviar(&self, mensagem: &str) {
        let mut emprestimo_um = self.mensagens_enviadas.borrow_mut();
        let mut emprestimo_dois = self.mensagens_enviadas.borrow_mut();

        emprestimo_um.push(String::from(mensagem));
        emprestimo_dois.push(String::from(mensagem));
    }
}
```

<span class="caption">Listagem 15-23: Criando duas referências mutáveis no mesmo
escopo para ver que o `RefCell<T>` irá "entrar em pânico" (i.e., executar
`panic!`)</span>

Nós criamos uma variável `emprestimo_um` para o ponteiro inteligente `RefMut<T>`
retornado por `borrow_mut`. Então criamos outro empréstimo mutável da mesma
forma na variável `emprestimo_dois`. Isto resulta em duas referências mutáveis
no mesmo escopo, o que não é permitido. Quando rodarmos os testes para nossa
biblioteca, o código na Listagem 15-23 irá compilar sem nenhum erro, mas o teste
irá falhar:

```text
---- tests::envia_uma_mensagem_de_aviso_de_acima_de_75_porcento stdout ----
	thread 'tests::envia_uma_mensagem_de_aviso_de_acima_de_75_porcento' entrou
    em pânico em
    'já emprestado: BorrowMutError', src/libcore/result.rs:906:4
nota: Rode com `RUST_BACKTRACE=1` para um backtrace.
```

Note como o código entrou em pânico com a mensagem `já emprestado: BorrowMutError`. É assim que o `RefCell<T>` lida com violações das regras de
empréstimo em tempo de execução.

Pegar erros de empréstimo em tempo de execução em vez de em tempo de compilação
significa encontrar defeitos no nosso código mais tarde no processo de
desenvolvimento, e possivelmente nem mesmo até que nosso código já tenha sido
implantado em produção. Além disso, nosso código irá incorrer em uma pequena
penalidade de desempenho de execução como resultado de manter registro dos
empréstimos em tempo de execução em vez de compilação. Ainda assim, usar o
`RefCell<T>` nos torna possível escrever um objeto simulado que pode se
modificar para registrar as mensagens que ele já viu enquanto o usamos em um
contexto onde apenas valores imutáveis são permitidos. Podemos usar o
`RefCell<T>`, apesar de seus trade-offs, para obter mais funcionalidade do que
referências regulares nos dão.

### Conseguindo Múltiplos Possuidores de Dados Mutáveis pela Combinação de `Rc<T>` e `RefCell<T>`

Um jeito comum de usar o `RefCell<T>` é em combinação com o `Rc<T>`. Lembre-se
de que o `Rc<T>` nos permite ter múltiplos possuidores de algum dado, mas ele só
nos permite acesso imutável a esse dado. Se temos um `Rc<T>` que contém um
`RefCell<T>`, podemos ter um valor que pode ter múltiplos possuidores _e_ que
podemos modificar!

Por exemplo, lembre-se da cons list na Listagem 15-18 onde usamos o `Rc<T>` para
nos permitir que múltiplas listas compartilhassem posse de outra lista. Como o
`Rc<T>` guarda apenas valores imutáveis, nós não podemos modificar nenhum dos
valores na lista uma vez que os criamos. Vamos adicionar o `RefCell<T>` para
ganhar a habilidade de mudar os valores nas listas. A Listagem 15-24 mostra que,
usando um `RefCell<T>` na definição do `Cons`, podemos modificar o valor
armazenado em todas as listas:

<span class="filename">Arquivo: src/main.rs</span>

```rust
#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}

use List::{Cons, Nil};
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    let valor = Rc::new(RefCell::new(5));

    let a = Rc::new(Cons(Rc::clone(&valor), Rc::new(Nil)));

    let b = Cons(Rc::new(RefCell::new(6)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(10)), Rc::clone(&a));

    *valor.borrow_mut() += 10;

    println!("a depois = {:?}", a);
    println!("b depois = {:?}", b);
    println!("c depois = {:?}", c);
}
```

<span class="caption">Listagem 15-24: Usando `Rc<RefCell<i32>>` para criar uma
`List` que podemos modificar</span>

Nós criamos um valor que é uma instância de `Rc<RefCell<i32>>` e o armazenamos
em uma variável chamada `valor` para que possamos acessá-lo diretamente mais
tarde. Então criamos uma `List` em `a` com uma variante `Cons` que guarda
`valor`.

Nós embrulhamos a lista `a` em um `Rc<T>` para que, quando criarmos as listas
`b` e `c`, elas possam ambas se referir a `a`, que é o que fizemos na Listagem
15-18.

Depois de criarmos as listas em `a`, `b` e `c`, adicionamos 10 ao valor em
`valor`. Fazemos isto chamando `borrow_mut` em `valor`, o que usa a
funcionalidade de desreferência automática que discutimos no Capítulo 5 (veja a
seção "Onde está o operador `->`?") para desreferenciar o `Rc<T>` ao valor
interno `RefCell<T>`. O método `borrow_mut` retorna um ponteiro inteligente
`RefMut<T>` no qual usamos o operador de desreferência e modificamos o valor
interno.

Quando imprimimos `a`, `b` e `c`, podemos ver que todos eles têm o valor
modificado de 15 em vez de 5:

```text
a depois = Cons(RefCell { value: 15 }, Nil)
b depois = Cons(RefCell { value: 6 }, Cons(RefCell { value: 15 }, Nil))
c depois = Cons(RefCell { value: 10 }, Cons(RefCell { value: 15 }, Nil))
```

Esta técnica é bem bacana! Usando um `RefCell<T>`, temos uma `List`
exteriormente imutável. Mas podemos usar os métodos no `RefCell<T>` que dão
acesso a sua mutabilidade interior para que possamos modificar nossos dados
quando precisarmos. As checagens em tempo de execução das regras de empréstimo
nos protegem de corridas de dados, e às vezes vale a pena trocar um pouco de
velocidade por esta flexibilidade nas nossas estruturas de dados.

A biblioteca padrão tem outros tipos que proveem mutabilidade interior, como o
`Cell<T>`, que é parecido, exceto que em vez de dar referências ao valor
interno, o valor é copiado para dentro e para fora do `Cell<T>`. Tem também o
`Mutex<T>`, que oferece mutabilidade interior que é segura de usar entre
threads; vamos discutir seu uso no Capítulo 16. Confira a documentação da
biblioteca padrão para mais detalhes sobre as diferenças entre estes tipos.
