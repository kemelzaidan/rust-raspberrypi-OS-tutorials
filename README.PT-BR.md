# Tutoriais de desenvolvimento de sistema operacional em Rust no Raspberry Pi

![](https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/workflows/BSP-RPi3/badge.svg) ![](https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/workflows/BSP-RPi4/badge.svg) ![](https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/workflows/Unit-Tests/badge.svg) ![](https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/workflows/Integration-Tests/badge.svg) ![](https://img.shields.io/badge/License-MIT%20OR%20Apache--2.0-blue)

<br/>

<img src="doc/header.jpg" height="372"> <img src="doc/minipush_demo_frontpage.gif" height="372">

## ℹ️ Introdução

Esta é uma série de tutoriais para desenvolvedores de sistemas operacionais amadores que são novos na [arquitetura ARMv8-A] de 64 bits da ARM. Os tutoriais fornecerão um "tour" guiado passo a passo de como escrever um `kernel` para um sistema operacional [monolítico] de um `sistema embarcado` do zero. Abrangendo a implementação de tarefas comuns de sistemas operacionais, como escrita no terminal serial, configuração de memória virtual e manipulação de exceções de hardware. Tudo isso aproveitando os recursos exclusivos do `Rust` para fornecer segurança e velocidade.

Divirta-se!

_Atenciosamente,<br>André ([@andre-richter])_

P.S.: Para outros idiomas, procure arquivos README alternativos. Por exemplo,
[`README.CN.md`](README.CN.md) ou [`README.ES.md`](README.ES.md). Muito obrigado ao nosso
[tradutores](#traduções-deste-repositório) 🙌.

[arquitetura ARMv8-A]: https://developer.arm.com/products/architecture/cpu-architecture/a-profile/docs
[monolítico]: https://pt.wikipedia.org/wiki/N%C3%BAcleo_monol%C3%ADtico
[@andre-richter]: https://github.com/andre-richter

## 📑 Organização

- Cada tutorial contém um binário `kernel` independente e inicializável.
- Cada novo tutorial amplia o anterior.
- Cada tutorial `README` terá uma curta seção `tl;dr` dando uma breve visão geral das adições,
   e mostre o código fonte `diff` do tutorial anterior, para que você possa inspecionar convenientemente o
   alterações/acréscimos.
     - Alguns tutoriais possuem um texto completo e detalhado além da seção `tl;dr`. O plano de longo prazo é que todos os tutoriais recebam um texto completo, mas por enquanto isso é exclusivo para
tutoriais onde acho que `tl;dr` e `diff` não são suficientes para se ter uma ideia.
- O código escrito nestes tutoriais é compatível e executado no **Raspberry Pi 3** e no **Raspberry Pi 4**.
   - Os tutoriais 1 a 5 são códigos básicos que só fazem sentido rodar em `QEMU`.
   - Começando com o [tutorial 5](05_drivers_gpio_uart), você pode carregar e executar o kernel nos Raspberrys reais e observar a saída em `UART`.
- Embora o Raspberry Pi 3 e 4 sejam os principais alvos, o código é escrito de forma modular de modo que permite fácil portabilidade para outras arquiteturas de CPU e/ou placas.
   - Eu realmente adoraria se alguém tentasse uma implementação no **RISC-V**!
- Para escrita, recomendo o [Visual Studio Code] com [Rust Analyzer].
- Além do texto do tutorial, confira também o comando `make doc` em cada tutorial. Isso permite a você navegar pelo código amplamente documentado de maneira conveniente.

### Saída do `make doc`

![fazer documento](doc/make_doc.png)

[Visual Studio Code]: https://code.visualstudio.com
[Rust Analyzer]: https://rust-analyzer.github.io

## 🛠 Requisitos de sistema

Os tutoriais são direcionados principalmente às distribuições baseadas em **Linux**. A maioria das coisas também funcionará no **macOS**, mas isso é apenas _experimental_.

### 🚀 A versão tl; dr

1. [Instalar Docker Engine][install_docker].
1. (**Somente Linux**) Certifique-se de que sua conta de usuário esteja no [grupo docker].
1. Prepare o _toolchain_ (conjunto de ferramentas) `Rust`. A maior parte será tratada na primeira utilização através do arquivo [rust-toolchain.toml](rust-toolchain.toml). O que nos resta fazer é:
    1. Se você já possui uma versão do Rust instalada:

       ```bash
       cargo install cargo-binutils rustfilt
       ```

    1. Se você precisar instalar o Rust do zero:

       ```bash
       curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

       source $HOME/.cargo/env
       cargo install cargo-binutils rustfilt
       ```

1. Caso você use o `Visual Studio Code`, recomendo fortemente a instalação da [extensão Rust Analyzer].
1. (**somente macOS**) Instale alguns `Ruby` gems.

   Isso foi testado pela última vez pelo autor com Ruby versão `3.0.2` no `macOS Monterey`. Se você estiver usando
   `rbenv`, o respectivo arquivo `.ruby-version` já está instalado. Se você nunca ouviu falar de `rbenv`,
   tente usar [este pequeno guia](https://stackoverflow.com/a/68118750).

    Execute isto na pasta raiz do repositório:

   ```bash
bundle config set --local path '.vendor/bundle'
bundle config set --local without 'development'
bundle install
	```

[grupo docker]: https://docs.docker.com/engine/install/linux-postinstall/
[extensão Rust Analyzer]: https://marketplace.visualstudio.com/items?itemName=matklad.rust-analyzer

### 🧰 Mais detalhes: Eliminando problemas com o _toolchain_

Esta série tenta colocar um forte foco na facilidade de uso. Portanto, foi feito o maior esforço possível para eliminar a maior fonte de "dor de cabeça" no desenvolvimento de dispositivos embarcados: o `toolchain`.

O próprio Rust já está ajudando muito nesse aspecto, pois possui suporte integrado para compilação cruzada (_cross-compilation_). Tudo o que precisamos para compilação cruzada de um _host_ `x86` para a arquitetura `AArch64` do Raspberry Pi será instalado automaticamente pelo `rustup`. Porém, além do compilador Rust, usaremos mais algumas ferramentas. Entre elas:

- `QEMU` para emular nosso kernel no sistema do _host_.
- Uma ferramenta feita por você mesmo chamada `Minipush` para carregar o kernel no Raspberry Pi sob demanda por meio do `UART`.
- `OpenOCD` e `GDB` para depuração no sistema embarcado.

Há muita coisa que pode dar errado durante a instalação e/ou compilação da versão correta de cada ferramenta na sua máquina host. Por exemplo, sua distribuição pode não fornecer a versão mais recente disponível que é necessária. Ou estão faltando algumas dependências difíceis de obter para a compilação de uma dessas ferramentas.

É por isso que usaremos [Docker][install_docker] sempre que possível. Estamos fornecendo um contêiner que possui todas as ferramentas ou dependências necessárias pré-instaladas e que é baixado automaticamente quando necessário. Se você quiser saber mais sobre o Docker e dê uma olhada no contêiner fornecido, consultando a pasta [docker](docker) do repositório.

[install_docker]: https://docs.docker.com/engine/install/#server

## 📟 Saída serial USB

Como o kernel desenvolvido nos tutoriais roda em um hardware real, é altamente recomendado que você tenha um cabo serial USB para obter a experiência completa.

- Você pode encontrar cabos USB-para-serial que devem funcionar sem problemas em [\[1\]] [\[2\]], mas muitos outros também funcionarão. Idealmente, seu cabo é baseado no chip `CP2102`.
- Você o conecta aos pinos `GND` e GPIO `14/15` conforme mostrado abaixo.
- O [tutorial 5](05_drivers_gpio_uart) é o primeiro onde você pode utilizá-lo. Confira as instruções sobre como preparar o cartão SD para inicializar o kernel feito por você a partir dele.
- Começando com o [tutorial 6](06_uart_chainloader), inicializar kernels no seu Raspberry começa a ficar _realmente_ confortável. Neste tutorial, é desenvolvido o chamado `chainloader`, que será o último arquivo que você precisa copiar manualmente no cartão SD por enquanto. Ele vai permitir que você carregue os kernels do tutorial sob demanda durante a inicialização utilizando o `UART`.

![Diagrama de fiação UART](doc/wiring.png)

[\[1\]]: https://www.amazon.de/dp/B0757FQ5CX/ref=cm_sw_r_tw_dp_U_x_ozGRDbVTJAG4Q
[\[2\]]: https://www.adafruit.com/product/954

## 🙌 Agradecimentos

A versão original dos tutoriais começou como um fork dos incríveis [tutoriais sobre programação "bare metal" no RPi3](https://github.com/bztsrc/raspi3-tutorial) em `C` do [Zoltan
Baldaszti](https://github.com/bztsrc). Obrigado por me dar o "pontapé inicial"!

### Traduções deste repositório

  - **Chinês**
    - [@colachg] e [@readlnh].
    - Precisa de atualização.
  - **Espanhol**
    - [@zanezhub].
    - Futuramente teremos tutoriais traduzidos para o espanhol.
  - **Português (BR)**
	- [@kemelzaidan]

[@colachg]: https://github.com/colachg
[@readlnh]: https://github.com/readlnh
[@zanezhub]: https://github.com/zanezhub
[@kemelzaidan]: https://github.com./kemelzaidan

## Licença

Licenciado sob qualquer uma das

- Licença Apache, Versão 2.0, ([LICENSE-APACHE](LICENSE-APACHE) ou http://www.apache.org/licenses/LICENSE-2.0)
- Licença MIT ([LICENSE-MIT](LICENSE-MIT) ou http://opensource.org/licenses/MIT)

a sua escolha.

### Contribuição

A menos que você declare explicitamente o contrário, qualquer contribuição feita por você e enviada intencionalmente para inclusão neste trabalho, conforme definido na licença Apache-2.0, deverá ter licenciamento duplo conforme acima, sem qualquer
termos ou condições adicionais.