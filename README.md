# Engenharia-Reversa-Formato-ELF-no-Raspberry-Pi-vol.1
*Formato ELF para Iniciantes - Análise prática no Raspberry Pi 4 (ARM64)*


> **Para quem é este guia?**
> Para qualquer pessoa curiosa sobre como os programas realmente funcionam por baixo dos panos, especialmente estudantes de Segurança, Engenharia Reversa e Sistemas Operacionais que tem um Raspberry Pi 4/5. Você não precisa saber nada de ELF para começar. Vamos construir o conhecimento juntos, tijolo por tijolo.


## Sumário

1. [Introdução](#1-introdução)
   - O que é um arquivo ELF?
   - Diferença entre ELF e PE (Windows)
   - Por que estudar ELF?
2. [Preparação do Ambiente no Raspberry Pi](#2-preparação-do-ambiente-no-raspberry-pi)
   - Instalando as ferramentas necessárias
   - 32-bit vs 64-bit: o que isso significa?
3. [Criação de um Executável ELF Simples](#3-criação-de-um-executável-elf-simples)
   - Escrevendo e compilando um "Hello World"
   - Gerando ELF 64-bit e 32-bit
4. [Análise Inicial do Arquivo ELF](#4-análise-inicial-do-arquivo-elf)
   - Estrutura básica: o ELF Header
   - Magic Number e campos essenciais
   - Diferenças entre ELF32 e ELF64
5. [Parte Prática com Editor Hex (ht)](#5-parte-prática-com-editor-hex-ht)
   - Instalando e navegando no `ht`
   - Analisando o ELF byte a byte


# 1. Introdução

## O que é um arquivo ELF?

Imagine que você escreveu uma receita de bolo em português. Para que outra pessoa consiga fazer o bolo, ela precisa:

1. Entender o **idioma** (português)
2. Saber a **ordem** dos ingredientes e dos passos
3. Ter os **utensílios** certos (forno, batedeira, etc.)

Um **executável** é como essa receita, mas para o computador. O problema é: cada sistema operacional "fala um idioma diferente" para cada tipo de executável. O **ELF** é o formato que o **Linux** (e sistemas Unix-like) usa para entender seus programas.

> **ELF = Executable and Linkable Format**
> (Formato Executável e Linkável)

Quando você digita `./meu_programa` no terminal Linux, o kernel lê o arquivo ELF, entende onde está o código, onde estão os dados, quais bibliotecas precisa carregar e só então começa a executar. O ELF é esse "contrato" entre o seu programa e o sistema operacional.

### ELF não serve só para executáveis!

O ELF é usado em três cenários principais:

```
┌────────────────────────────────────────────────────┐
│                   TIPOS DE ARQUIVO ELF                      │
├──────────────┬────────────────┬────────────────────┤
│  ET_EXEC        │  ET_DYN           │  ET_REL               │
│  Executável     │  Biblioteca (.so) │  Objeto (.o)          │
│  (./programa)   │  (libssl.so.3)    │  (main.o)             │
│                 │  OU PIE exec      │                       │
└──────────────┴────────────────┴────────────────────┘
```

- **ET_EXEC**: O programa que você executa diretamente
- **ET_DYN**: Bibliotecas compartilhadas (`.so`) (código reutilizável) e também executáveis modernos compilados como PIE (Position Independent Executable)
- **ET_REL**: Arquivos objeto, resultado da compilação antes do link


## Diferença entre ELF e PE (Windows)

Se você já usou Windows, talvez tenha ouvido falar de arquivos `.exe` ou `.dll`. Eles usam o formato **PE (Portable Executable)**. Ambos resolvem o mesmo problema: "como o SO deve carregar e executar esse programa?", mas de formas diferentes.

```
┌────────────────────────────────────────────────────────┐
│                   ELF  vs  PE  (visão geral)                     │
├────────────────────┬───────────────────────────────────┤
│        ELF             │              PE                         │
├────────────────────┼───────────────────────────────────┤
│ Linux, BSD, Android,   │ Windows (e alguns sistemas embarcados)  │
│ PS4/PS5, Switch...     │                                         │
├────────────────────┼───────────────────────────────────┤
│ Magic: 7F 45 4C 46     │ Magic: 4D 5A ("MZ") + "PE\0\0"          │
│ (7F E L F)             │                                         │
├────────────────────┼───────────────────────────────────┤
│ Seções + Segmentos     │ Seções organizadas em um único bloco    │
│ (dois "mapas" no       │ com um cabeçalho NT                     │
│ mesmo arquivo)         │                                         │
├────────────────────┼───────────────────────────────────┤
│ Bibliotecas: .so       │ Bibliotecas: .dll                       │
│ (shared objects)       │ (dynamic-link libraries)                │
├────────────────────┼───────────────────────────────────┤
│ Linker: ld, ld.so      │ Linker: link.exe, loader do ntdll.dll   │
└────────────────────┴─────────-–  ──────────────────────┘
```

> **Analogia**: Pense no ELF e no PE como dois tipos de passaporte. Os dois servem para "entrar em um país" (ser executado pelo SO), mas têm campos diferentes, tamanhos diferentes e o funcionário da imigração (kernel) sabe ler apenas o passaporte do seu próprio país.


## Por que estudar ELF?

Ótima pergunta. Aqui eu listo algumas razões práticas e diretas:

### Para Segurança Ofensiva (Red Team / Pentest)
- Entender como **malwares** escondem código (seções extras, headers corrompidos)
- Criar **exploits** que funcionam em binários reais
- Fazer **injeção de código** em processos Linux
- Bypassar proteções como **PIE, RELRO, NX/DEP**

### Para Segurança Defensiva (Blue Team / Análise de Malware)
- Analisar binários suspeitos sem executá-los (**análise estática**)
- Entender o que um malware tenta fazer ao modificar tabelas ELF
- Detectar **packers** e **obfuscação** em binários Linux

### Para Desenvolvimento e Debugging
- Entender por que seu programa crasha (seções, símbolos)
- Otimizar binários, remover símbolos de debug
- Trabalhar com **sistemas embarcados** (ARM, MIPS, RISC-V)

### Para o Raspberry Pi especificamente
- O Pi roda **ARM64 Linux**, então tudo é ELF aqui!
- Entender o firmware e o bootloader
- Analisar aplicações IoT e dispositivos embarcados


# 2. Preparação do Ambiente no Raspberry Pi

## Instalando as ferramentas necessárias

Abra o terminal do seu Raspberry Pi e vamos instalar tudo de uma vez. Esses comandos funcionam no **Raspberry Pi OS (baseado em Debian/Ubuntu)**.
*Eu estou usando uma VM (virt-manager) para caso ocorra qualquer problema ao longo desse estudo*

### Passo 1 — Atualizar o sistema

```bash
sudo apt update && sudo apt upgrade -y
```

> Cuidado nunca é demais, né...


### Passo 2 — Instalar as ferramentas principais

```bash
# Compilador C e ferramentas de build
sudo apt install -y gcc make

# Nota: gcc-multilib é um pacote x86/x86_64 e NÃO existe no Raspberry Pi OS ARM64.
# Para suporte a 32-bit no ARM64, usamos cross-compiler (explicado mais adiante).

# Ferramentas de análise binária (binutils)
sudo apt install -y binutils

# hexdump (geralmente já instalado, mas garantindo)
sudo apt install -y bsdmainutils

# file: identifica tipos de arquivo
sudo apt install -y file

# xxd: outro visualizador hex (mais amigável que hexdump às vezes)
sudo apt install -y xxd

# ht: editor hex avançado com modo visual no terminal
sudo apt install -y ht

# ltrace e strace: rastrear chamadas de sistema e biblioteca
sudo apt install -y strace ltrace

# gdb: debugger GNU
sudo apt install -y gdb

# Opcional mas muito útil: pwndbg (extensão do GDB)
# sudo apt install -y python3-pip
# pip3 install pwntools
```


### Passo 3 — Verificar a instalação

```bash
# Ferramentas que suportam --version
gcc --version
readelf --version
objdump --version
file --version

# hexdump e ht não suportam --version; use which para confirmar instalação
which hexdump
which hte
```

Você deve ver saídas parecidas com estas:

```
gcc (Debian 14.2.0-19) 14.2.0
...
GNU readelf version 2.44
...
GNU objdump version 2.44
...
file-5.46
...
/usr/bin/hexdump
/usr/bin/hte
```

> Se todos os comandos retornaram informações sem erro, você está pronto!


### Tabela de ferramentas e suas funções

```
┌──────────── ─┬────────────────────────────────────────────┐
│  Ferramenta    │  Para que serve                                    │
├──────────── ─┼────────────────────────────────────────────┤
│  gcc           │  Compilador C — transforma .c em binário ELF       │
│  readelf       │  Lê e exibe informações do arquivo ELF             │
│  objdump       │  Desmonta (disassembly) e inspeciona ELF           │
│  nm            │  Lista símbolos (funções, variáveis) do binário    │
│  strings       │  Extrai strings legíveis do binário                │
│  file          │  Identifica o tipo do arquivo                      │
│  hexdump / xxd │  Exibe conteúdo em hexadecimal                     │
│  ht            │  Editor hex interativo no terminal                 │
│  strace        │  Rastreia syscalls feitas pelo programa            │
│  ltrace        │  Rastreia chamadas a bibliotecas (.so)             │
│  gdb           │  Debugger: executa passo a passo, inspeção         │
└───────────── ┴────────────────────────────────────────────┘
```


## 32-bit vs 64-bit — O que isso significa e por que importa?

### Verificando seu sistema...

```bash
# Verificar arquitetura do processador
uname -m

# Verificar se é 64-bit
getconf LONG_BIT

# Ver arquitetura do kernel
arch
```

No Raspberry Pi 4 com OS de 64 bits, você verá:

```
$ uname -m
aarch64

$ getconf LONG_BIT
64

$ arch
aarch64
```

> **aarch64** é a mesma coisa que **ARM64**.


### O que muda entre 32-bit e 64-bit?

Pense assim: um registrador de processador é como uma "caixa" onde o CPU guarda números temporariamente.

```
Processador 32-bit:
┌───────────────────────────────────────────────────────────────────────────┐
│              R0 (32 bits)                         │  ← Cabe números até 4.294.967.295  
│  [00000000 00000000 00000000 00000000]            │    (2³² - 1)                        
└───────────────────────────────────────────────────────────────────────────┘

Processador 64-bit:
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                           X0 (64 bits)                                      │  Cabe números muito maiores
│  [00000000 00000000 00000000 00000000 | 00000000 00000000 00000000 00000000 │ ←
│                                                                             │  E endereços de memória maiores!
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

**Consequências práticas no ELF:**

| Característica           | ELF 32-bit (ARM)     | ELF 64-bit (AArch64)     |
|--------------------------|----------------------|--------------------------|
| Endereços de memória     | 4 bytes (32 bits)    | 8 bytes (64 bits)        |
| Espaço de memória        | Até 4 GB             | Até 16 Exabytes (teórico)|
| Nome do campo `e_machine`| `EM_ARM (40)`        | `EM_AARCH64 (183)`       |
| Tamanho do ELF Header    | 52 bytes             | 64 bytes                 |
| Tamanho do Phdr          | 32 bytes             | 56 bytes                 |
| Tamanho do Shdr          | 40 bytes             | 64 bytes                 |
| Instrução típica         | `MOV R0, #1`         | `MOV X0, #1`             |
| Registradores            | R0–R15               | X0–X30                   |

### Quando usar 32-bit vs 64-bit?

```
Compile para 64-bit (aarch64) quando:
  - Desenvolvendo para sistemas modernos
  - Precisar de mais de 4GB de RAM endereçável
  - Quiser melhor performance no Raspberry Pi 4
  - For o padrão do seu SO (Raspberry Pi OS 64-bit)

Compile para 32-bit (arm) quando:
  - Analisando malware ou binários antigos de 32-bit
  - Testando compatibilidade
  - Trabalhando com dispositivos IoT mais antigos
  - Estudar diferenças arquiteturais (fins didáticos)
  Obs: Requer bibliotecas de 32-bit instaladas no Pi
```


### Instalando suporte a 32-bit no Raspberry Pi OS 64-bit

Se quiser compilar binários ARM 32-bit no seu Pi 4 64-bit:

```bash
# Habilitar arquitetura armhf
sudo dpkg --add-architecture armhf
sudo apt update

# Instalar compilador cruzado e bibliotecas 32-bit
sudo apt install -y gcc-arm-linux-gnueabihf
sudo apt install -y libc6:armhf

# Verificar instalação
arm-linux-gnueabihf-gcc --version
```

> **Nota**: No ARM, "cross-compile" para 32-bit é diferente do x86 onde você usa `-m32`. No ARM64, você usa um compilador específico para ARM32 (`arm-linux-gnueabihf-gcc`). Falaremos mais sobre isso na seção seguinte.


# 3. Criação de um Executável ELF Simples

## Escrevendo o programa Hello World em C

Vamos criar o programa mais clássico da programação!!!
No terminal (usuário comum):

```bash
# Criar um diretório para nossos experimentos
mkdir -p ~/estudos_elf
cd ~/estudos_elf

# Criar o arquivo C
nano hello.c
```

Digite o seguinte código:

```
/*
 * hello.c — Programa de estudo do formato ELF
 * Autor: você mesmo!
 *
 * Este programa simples vai nos gerar um belo ELF para analisarmos em detalhes.
 */

#include <stdio.h>

int main(void) {
    printf("Olá, ELF! Estou aprendendo Engenharia Reversa!\n");
    return 0;
}
```

Salve com `Ctrl+O`, `Enter`, e saia com `Ctrl+X`.


## Compilando para 64-bit (AArch64) — o padrão do Raspberry Pi 4

```bash
# Compilação padrão (64-bit no Raspberry Pi OS 64-bit)
gcc -o hello_64 hello.c

# Verificar o que foi gerado
file hello_64
```

Saída esperada:

```
hello_64: ELF 64-bit LSB pie executable, ARM aarch64, version 1 (SYSV),
dynamically linked, interpreter /lib/ld-linux-aarch64.so.1,
BuildID[sha1]=..., for GNU/Linux 3.7.0, not stripped
```

> Você acabou de criar seu primeiro ELF para ARM64! Emocionante, não?!


### Entendendo as flags de compilação importantes

```bash
# Compilação básica (como fizemos acima)
gcc -o hello_64 hello.c

# Com símbolos de debug (ótimo para análise!)
gcc -g -o hello_64_debug hello.c

# Sem otimizações (mais fácil de analisar)
gcc -O0 -o hello_64_O0 hello.c

# Com otimizações máximas (mais difícil de reverter)
gcc -O2 -o hello_64_O2 hello.c

# Sem PIE (gera ET_EXEC em vez de ET_DYN — endereços fixos)
gcc -no-pie -o hello_64_nopie hello.c

# Linkado estaticamente (tudo incluído no binário, sem .so)
gcc -static -o hello_64_static hello.c

# Removendo símbolos (como malware faz para dificultar análise)
gcc -s -o hello_64_stripped hello.c
```

#### Tabela de flags e seus efeitos

```
┌──────────────────┬────────────────────────────────────────────┐
│  Flag               │  Efeito no binário ELF resultante                  │
├──────────────────┼────────────────────────────────────────────┤
│  (padrão)           │  PIE habilitado, dinâmico, sem debug               │
│  -g                 │  Adiciona seção .debug_* com info de debug         │
│  -O0                │  Sem otimização: código mais legível no reversing  │
│  -O2 / -O3          │  Otimizado: inline, loop unroll, mais difícil      │
│  -no-pie            │  Endereço base fixo — mais fácil de analisar       │
│  -static            │  Binário enorme, mas independente (sem .so)        │
│  -s                 │  Remove tabela de símbolos (strip)                 │
│  -fno-stack-protector│  Remove canário de stack (útil para CTF/exploits) │
│  -Wl,-z,execstack   │  Stack executável (remove NX — PERIGOSO em prod.)  │
└──────────────────┴────────────────────────────────────────────┘
```


## Gerando ELF 32-bit (ARM) no Raspberry Pi 4

Como mencionamos, no ARM64 precisamos do compilador cruzado:

```bash
# Usando o cross-compiler ARM 32-bit
arm-linux-gnueabihf-gcc -o hello_32 hello.c

# Verificar
file hello_32
```

Saída esperada:

```
hello_32: ELF 32-bit LSB pie executable, ARM, EABI5 version 1 (SYSV),
dynamically linked, interpreter /lib/ld-linux-armhf.so.3,
BuildID[sha1]=..., for GNU/Linux 3.2.0, not stripped
```

> **Nota**: O `hello_32` **não vai executar diretamente** no Raspberry Pi OS puro de 64-bit sem as libs de 32-bit instaladas. Se instalou o suporte a armhf como mostrado na seção anterior, você pode executar normalmente.


## Comparando os dois binários

> A partir daqui eu vou parecer um pouco repetitivo com alguns comandos, mas meu objetivo é que você se familiarize e realmente entenda o que é cada um e o que eles nos dizem. Continuando...

```bash
# Tamanho dos binários
ls -lh hello_64 hello_32

# Comparar com file
file hello_64 hello_32
```

Exemplo de saída:

```
-rwxr-xr-x 1 pi pi 8.0K  hello_64
-rwxr-xr-x 1 pi pi 7.1K  hello_32

hello_64: ELF 64-bit LSB pie executable, ARM aarch64 ...
hello_32: ELF 32-bit LSB pie executable, ARM, EABI5 ...
```

> O binário 64-bit é ligeiramente maior porque os endereços e ponteiros ocupam 8 bytes em vez de 4 bytes.


# 4. Análise Inicial do Arquivo ELF 

## Usando o comando `file`

Sempre comece com `file`. É o "documento de identidade" do binário:

```bash
file hello_64
file hello_32
```

```
hello_64: ELF 64-bit LSB pie executable, ARM aarch64, version 1 (SYSV),
          dynamically linked, interpreter /lib/ld-linux-aarch64.so.1,
          BuildID[sha1]=a1b2c3d4e5f6..., for GNU/Linux 3.7.0, not stripped

hello_32: ELF 32-bit LSB pie executable, ARM, EABI5 version 1 (SYSV),
          dynamically linked, interpreter /lib/ld-linux-armhf.so.3,
          BuildID[sha1]=f6e5d4c3b2a1..., for GNU/Linux 3.2.0, not stripped
```

Decodificando cada campo:

```
┌────────────────────────────────────────────────────────────┐
│  Decodificando a saída do comando "file"                             │
├─────────────────────┬──────────────────────────────────── ─┤
│  "ELF"                 │  Formato: Executable and Linkable Format    │
│  "64-bit"              │  Classe: ELF64                              │
│  "LSB"                 │  Endianness: Little-Endian (Intel/ARM)      │
│  "pie executable"      │  Tipo: Position Independent Executable      │
│  "ARM aarch64"         │  Arquitetura: ARM de 64 bits                │
│  "version 1 (SYSV)"    │  Versão ELF + ABI: Unix System V            │
│  "dynamically linked"  │  Usa bibliotecas .so externas               │
│  "interpreter ..."     │  Caminho do dynamic linker                  │
│  "BuildID[sha1]=..."   │  Hash único desse build                     │
│  "not stripped"        │  Ainda tem tabela de símbolos               │
└─────────────────────┴──────────────────────────────────────┘
```


## Estrutura do ELF — A Grande Foto

Antes de mergulhar nos bytes, vamos entender a estrutura geral. Pense no ELF como um livro organizado:

```
┌─────────────────────────────────────────────────────┐
│                   ESTRUTURA DO ARQUIVO ELF                    
│                                                               
│  ┌───────────────────────────────────────────────┐    
│  │       ELF HEADER            │  ← "Capa do livro":         
│  │  (52 bytes em 32-bit /      │    tipo, arquitetura,        
│  │   64 bytes em 64-bit)       │    onde estão as outras      
│  │                             │    estruturas                
│  └───────────────────────────────────────────────┘   
│                                                               
│  ┌───────────────────────────────────────────────┐   
│  │  PROGRAM HEADER TABLE       │  ← "Índice para o SO":      
│  │  (opcional em .o)           │    segmentos para           
│  │  Lista de Segmentos         │    carregar na m emória     
│  └───────────────────────────────────────────────┘   
│                                                               
│  ┌────────────────────────────────────────────────┐  
│  │  SEÇÕES (Sections)          │  ← "Capítulos do livro":    
│  │  .text   (código)           │    código, dados,           
│  │  .data   (dados init.)      │    strings, símbolos...     
│  │  .rodata (dados readonly)   │                            
│  │  .bss    (dados não-init.)  │                             
│  │  .plt    (trampolim .so)    │                             
│  │  .got    (endereços .so)    │                             
│  │  .symtab (símbolos)         │                             
│  │  .strtab (nomes)            │                             
│  │  ... e muitas outras        │                             
│  └────────────────────────────────────────────────┘  
│                                                               
│  ┌────────────────────────────────────────────────┐  
│  │  SECTION HEADER TABLE       │  ← "Índice para o linker"   
│  │  (opcional em executáveis   │    descreve cada seção      
│  │   pós-strip)                │                             
│  └────────────────────────────────────────────────┘   
│                                                              
└─────────────────────────────────────────────────────┘

Nota: Segmentos (Program Headers) são usados em TEMPO DE EXECUÇÃO
      Seções (Section Headers) são usadas em TEMPO DE LINK/DEBUG
      Um segmento pode conter várias seções!
```


## Lendo o ELF Header com `readelf -h`

Este é o comando mais importante para começar:

```bash
readelf -h hello_64
```

Saída completa comentada:

```
ELF Header:
Magic: 7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00

| Posição | Byte   | Campo          | Valor      | Significado                       
|---------|--------|----------------|------------|-----------------------------------
| 0       | 7F     | EI_MAG0        | 0x7F       | Início do Magic Number              
| 1       | 45     | EI_MAG1        | 'E'        | Parte de "ELF"                                   
| 2       | 4C     | EI_MAG2        | 'L'        | Parte de "ELF"                       
| 3       | 46     | EI_MAG3        | 'F'        | Parte de "ELF"                                   
| 4       | 02     | EI_CLASS       | 2          | **ELF64**                                        
| 5       | 01     | EI_DATA        | 1          | Little endian (2's complement)          
| 6       | 01     | EI_VERSION     | 1          | Versão atual                         
| 7       | 00     | EI_OSABI       | 0          | UNIX - System V                                  
| 8       | 00     | EI_ABIVERSION  | 0          | ABI Version (nenhuma)               


 Informações Gerais

| Campo                                   | Valor                                      | Descrição 
|-----------------------------------------|--------------------------------------------|----------------------------
| Class                                   | ELF64                                      | 64-bit 
| Data                                    | 2's complement, little endian              | Little Endian 
| Version                                 | 1 (current)                                | Versão atual 
| OS/ABI                                  | UNIX - System V                            | Sistema alvo 
| ABI Version                             | 0                                          | Sem versão específica 
| Type                                    | DYN (Position-Independent Executable file) | Executável PIE 
| Machine                                 | AArch64                                    | ARM 64-bit (Raspberry Pi 4)
| Entry point address                     | 0x640                                      | Endereço de entrada 
| Start of program headers                | 64 bytes                                   | Início dos Program Headers 
| Start of section headers                | 6688 bytes                                 | Início dos Section Headers 
| Number of program headers               | 9                                          | Quantidade de PH 
| Number of section headers               | 27                                         | Quantidade de SH 
| Section header string table index       | 26                                         | Tabela de nomes das seções 


```

### Analisando campo por campo

```
┌───────────────────────────────────────────────────────────────┐
│  MAPA DO ELF HEADER (64-bit)                                             
├───────┬───────────────────┬───────┬───────────────────────────┤
│ Offset │ Campo                │ Tamanho│ Valor / Significado             
├───────┼──────────────────┼───────┼────────────────────────────┤
│ 0x00   │ e_ident[EI_MAG]      │ 4 bytes│ 7F 45 4C 46 = "\x7FELF"         │
│ 0x04   │ e_ident[EI_CLASS]    │ 1 byte │ 02 = ELF64 (01=ELF32)           │
│ 0x05   │ e_ident[EI_DATA]     │ 1 byte │ 01 = LSB/Little-Endian          │
│        │                      │        │ 02 = MSB/Big-Endian             │
│ 0x06   │ e_ident[EI_VERSION]  │ 1 byte │ 01 = versão atual               │
│ 0x07   │ e_ident[EI_OSABI]    │ 1 byte │ 00 = Unix System V              │
│ 0x08   │ e_ident[EI_ABIVERSION│ 1 byte │ 00 = não especificado           │
│ 0x09   │ e_ident[EI_PAD]      │ 7 bytes│ 00 00 00 00 00 00 00 (padding)  │
│ 0x10   │ e_type               │ 2 bytes│ 03 00 = ET_DYN (PIE exec)       │
│ 0x12   │ e_machine            │ 2 bytes│ B7 00 = EM_AARCH64 (183)        │
│ 0x14   │ e_version            │ 4 bytes│ 01 00 00 00                     │
│ 0x18   │ e_entry              │ 8 bytes│ Entry point (onde execução      │
│        │                      │        │ começa)                         │
│ 0x20   │ e_phoff              │ 8 bytes│ Offset da Program Header Table  │
│ 0x28   │ e_shoff              │ 8 bytes│ Offset da Section Header Table  │
│ 0x30   │ e_flags              │ 4 bytes│ Flags da arquitetura            │
│ 0x34   │ e_ehsize             │ 2 bytes│ 40 00 = 64 bytes (tamanho EH)   │
│ 0x36   │ e_phentsize          │ 2 bytes│ 38 00 = 56 bytes por entry Phdrn│
│ 0x38   │ e_phnum              │ 2 bytes│ Número de Program Headers       │
│ 0x3A   │ e_shentsize          │ 2 bytes│ 40 00 = 64 bytes por entry Shdr │
│ 0x3C   │ e_shnum              │ 2 bytes│ Número de Section Headers       │
│ 0x3E   │ e_shstrndx           │ 2 bytes│ Índice da seção de nomes        │
└───────┴───────────────────┴───────┴───────────────────────────┘
```


## O Magic Number: `7F 45 4C 46`

Este é o "cartão de identidade" de qualquer arquivo ELF. Os primeiros 4 bytes de **TODO** arquivo ELF no mundo são sempre:

```
Byte 0:  0x7F → Não imprimível (DEL no ASCII)
Byte 1:  0x45 → 'E'
Byte 2:  0x4C → 'L'
Byte 3:  0x46 → 'F'
```

```
┌─────────────────────────────────────────────┐
│           Magic Numbers de Formatos Comuns          │
├───────────────┬──────────────────────────────
│  Formato        │  Magic Bytes (hex)                │
├───────────────┼──────────────────────────────
│  ELF            │  7F 45 4C 46  ("\x7FELF")         │
│  PE (Windows)   │  4D 5A        ("MZ")              │
│  Mach-O (macOS) │  CF FA ED FE  (64-bit)            │
│  ZIP / JAR / APK│  50 4B 03 04  ("PK\x03\x04")      │
│  PDF            │  25 50 44 46  ("%PDF")            │
│  PNG            │  89 50 4E 47  ("\x89PNG")         │
│  Java .class    │  CA FE BA BE  (0xCAFEBABE)        │
└───────────────┴─────────────────────────────┘
```

> **Dica de Reversing**: Sempre que você encontrar um arquivo suspeito sem extensão, os primeiros bytes revelam seu verdadeiro tipo. Um malware pode renomear um ELF para `.txt`, mas os magic bytes `7F 45 4C 46` não mentem!


## Visualizando com `hexdump -C`

```bash
# Ver os primeiros 64 bytes (= tamanho do ELF Header de 64-bit)
hexdump -C hello_64 | head -5
```

Saída:

```
00000000  7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00  |.ELF............|
00000010  03 00 b7 00 01 00 00 00  40 06 00 00 00 00 00 00  |........@.......|
00000020  40 00 00 00 00 00 00 00  20 1a 00 00 00 00 00 00  |@....... .......|
00000030  00 00 00 00 40 00 38 00  09 00 40 00 1b 00 1a 00  |....@.8...@.....|
```

Mapeando esses bytes para o ELF Header:

```
Offset   Bytes                            Campo
─────   ────────────────────────    ──────────────────────
0x00     7f 45 4c 46                      Magic Number (\x7FELF)
0x04     02                               Classe: ELF64
0x05     01                               Endianness: LSB
0x06     01                               Versão ELF
0x07     00                               OS/ABI: Unix System V
0x08     00                               EI_ABIVERSION: 0 (não especificado)
0x09     00 00 00 00 00 00 00             EI_PAD: padding (7 bytes zeros)
0x10     03 00                            e_type: ET_DYN
0x12     b7 00                            e_machine: EM_AARCH64 (0xB7 = 183)
0x14     01 00 00 00                      e_version: 1
0x18     40 06 00 00 00 00 00 00          e_entry: 0x640
0x20     40 00 00 00 00 00 00 00          e_phoff: 64 (offset Phdr)
0x28     20 1a 00 00 00 00 00 00          e_shoff: 6688 (offset Shdr)
0x30     00 00 00 00                      e_flags: 0
0x34     40 00                            e_ehsize: 64 bytes
0x36     38 00                            e_phentsize: 56 bytes
0x38     09 00                            e_phnum: 9
0x3A     40 00                            e_shentsize: 64 bytes
0x3C     1b 00                            e_shnum: 27
0x3E     1a 00                            e_shstrndx: 26
```

> **Atenção**: O campo `EI_ABIVERSION` (offset `0x08`) **não é padding** — é um campo próprio da spec ELF que indica a versão da ABI. Quase sempre vale `0x00`, por isso parece "só mais um zero", mas tem um nome e significado distintos. O padding real começa no offset `0x09` e vai até `0x0F` (7 bytes).

> **Exercício mental**: O campo `e_machine` vale `b7 00`. Como estamos em Little-Endian, lemos ao contrário: `00 B7` = `0x00B7` = **183 decimal**. E `EM_AARCH64 = 183`. Tudo bate!


## Diferenças estruturais entre ELF32 e ELF64

```bash
# Comparar headers dos dois binários
readelf -h hello_64
readelf -h hello_32   # (se compilou o 32-bit)
```

```
┌──────────────────────────────────────────────────────────┐
│           ELF32  vs  ELF64 — Diferenças no Header                  │
├───────────────────┬──────────────────┬───────────────────
│  Campo               │  ELF32              │  ELF64                │
├───────────────────┼──────────────────┼───────────────────
│  e_ident[EI_CLASS]   │  01 (ELFCLASS32)    │  02 (ELFCLASS64)      │
│  e_machine           │  28 00 (ARM=40)     │  B7 00 (AArch64=183)  │
│  e_entry             │  4 bytes            │  8 bytes              │
│  e_phoff             │  4 bytes            │  8 bytes              │
│  e_shoff             │  4 bytes            │  8 bytes              │
│  e_ehsize            │  52 bytes           │  64 bytes             │
│  e_phentsize         │  32 bytes           │  56 bytes             │
│  e_shentsize         │  40 bytes           │  64 bytes             │
├───────────────────┴──────────────────┴───────────────────
│  Por que maior no 64-bit? Porque endereços e offsets são 8 bytes   │
│  em vez de 4 bytes, então todas as estruturas crescem.             │
└──────────────────────────────────────────────────────────┘
```


## Listando as Seções com `readelf -S`

```bash
readelf -S --wide hello_64
```

Saída (resumida):

```
There are 27 section headers, starting at offset 0x1a20:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        0000000000000238 000238 00001b 00   A  0   0  1
  [ 2] .note.gnu.build-id NOTE           0000000000000254 000254 000024 00   A  0   0  4
  ...
  [12] .text             PROGBITS        0000000000000580 000580 0001a4 00  AX  0   0 16
  [13] .rodata           PROGBITS        0000000000000724 000724 000028 00   A  0   0  8
  ...
  [23] .symtab           SYMTAB          0000000000000000 001488 0006a8 18     24  49  8
  [24] .strtab           STRTAB          0000000000000000 001b30 000362 00      0   0  1
  [25] .shstrtab         STRTAB          0000000000000000 001e92 0000f9 00      0   0  1
```

### As seções mais importantes:

```
┌────────────┬────┬───────────────────────────────────────────┐
│  Seção       │Flag │  Conteúdo                                         │
├────────────┼────┼────────────────────────────────────────────
│  .text       │ AX  │  Código executável do programa                    │
│  .data       │ WA  │  Variáveis globais inicializadas                  │
│  .rodata     │  A  │  Dados somente-leitura (strings "Hello World!")   │
│  .bss        │ WA  │  Variáveis globais não-inicializadas (= zeros)    │
│  .plt        │ AX  │  Procedure Linkage Table (trampolim para .so)     │
│  .got        │ WA  │  Global Offset Table (endereços de .so)           │
│  .symtab     │     │  Tabela de símbolos (nomes de funções/vars)       │
│  .strtab     │     │  Strings dos nomes na .symtab                     │
│  .shstrtab   │     │  Strings dos nomes das próprias seções            │
│  .interp     │  A  │  Caminho do dynamic linker                        │
│  .dynamic    │ WA  │  Informações de linking dinâmico                  │
├────────────┴────┴────────────────────────────────────────────
│  Flags: A=Allocable (ocupa memória), X=eXecutable, W=Writable          │
└─────────────────────────────────────────────────────────────┘
```

---

## Verificando os símbolos com `nm`

```bash
nm hello_64
```

```
...
0000000000000754 T main
                 U puts@GLIBC_2.17
...
```

- `T` = função na seção `.text` (código)
- `U` = símbolo indefinido (vem de uma biblioteca externa)
- `@GLIBC_2.17` = versão mínima da libc necessária para esse símbolo

> **Nota**: O GCC geralmente otimiza `printf("texto\n")` — sem variáveis — para `puts("texto")`, que é mais eficiente. Por isso `puts` aparece em vez de `printf`. Para forçar `printf`, compile com `-O0` ou use um formato com variável: `printf("Olá %s\n", "mundo")`.
>
> **Endereço de `main` nunca é `0x0000` num executável!** O endereço `0000...0000` aparece em arquivos objeto (`.o`), antes do link. Num executável ELF, `main` sempre tem um endereço real (como `0x754` acima).


## Extraindo strings com `strings`

```bash
strings hello_64
```

Você verá algo como:

```
/lib/ld-linux-aarch64.so.1
libc.so.6
__libc_start_main
...
Olá, ELF! Estou aprendendo Engenharia Reversa!
...
GCC: (Debian 12.2.0-14) 12.2.0
```

> **Dica de reversing**: Em malwares, `strings` revela IPs, URLs, mensagens, nomes de arquivos e muito mais — sem nem precisar executar o binário!


# 5. Parte Prática com Editor Hex (ht)

## O que é o `ht`?

O `ht` (HT Editor) é um editor hexadecimal e disassembler que roda **direto no terminal**. Diferente de outros hex editors, ele consegue entender o formato ELF e navegar pelas estruturas.

```bash
# Verificar se está instalado
which ht

# Se não estiver:
sudo apt install -y ht
```

---

## Abrindo um binário ELF no `ht`

```bash
# Ir para o diretório dos nossos binários
cd ~/estudos_elf

# Abrir o hello_64 no ht
ht hello_64
```

Você verá uma tela assim (adaptada para texto):

```
┌───────────────────────────────────────────────────────────┐
│ HT Editor - hello_64                                       [F1=Help]│
├──────────────────────────────────────────────────────── ──
│ 00000000  7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00 │.ELF....│
│ 00000010  03 00 b7 00 01 00 00 00  40 06 00 00 00 00 00 00 │........│
│ 00000020  40 00 00 00 00 00 00 00  20 1a 00 00 00 00 00 00 │@.......│
│ 00000030  00 00 00 00 40 00 38 00  09 00 40 00 1b 00 1a 00 │....@.8.│
├───────────────────────────────────────────────────────────
│ [F1]Help [F2]Save [F4]Hex [F5]Goto [F6]Find [F8]Analyze [F10]Quit   │
└──────────────────────────────────────────────── ──────────┘
```


## Comandos essenciais do `ht`

```
┌──────────────────────────────────────────────────────┐
│  ATALHOS DO HT EDITOR                                          │
├──────────────┬────────────────────────────────────────
│  Tecla       │  Ação                                           │
├──────────────┼───────────────────────────────────────
│  F1          │  Ajuda                                          │
│  F4          │  Trocar modo (hex / texto / desassembly)        │
│  F5          │  Ir para offset (ex: 0x10 → vai ao byte 0x10)  │
│  F6          │  Buscar bytes ou texto                          │
│  F8          │  Analisar estruturas ELF                        │
│  Alt+F4      │  Fechar arquivo                                 │
│  F10 / q     │  Sair                                           │
│  Setas       │  Navegar byte a byte                            │
│  PgUp/PgDn   │  Navegar página a página                        │
│  Tab         │  Alternar entre painel hex e painel texto       │
└──────────────┴───────────────────────────────────────┘
```


## Analisando o ELF byte a byte no `ht`

### Exercício 1 — Identificar o Magic Number

Ao abrir o arquivo, você já está no offset `0x00`. Os primeiros 4 bytes são:

```
Offset    Hex              ASCII
00000000  7f 45 4c 46 ...  .ELF
```

- `7F` → não tem caractere visível (é DEL no ASCII), representado como `.`
- `45` → `'E'`
- `4C` → `'L'`
- `46` → `'F'`

> Confirmado: é um ELF!


### Exercício 2 — Identificar a classe (32 ou 64 bit)

Pressione `F5` e digite `4` para ir ao offset `0x04`:

```
Offset 0x04:  02
```

- `01` = ELF32
- `02` = ELF64

Nosso arquivo tem `02` → **ELF64**.


### Exercício 3 — Identificar o endianness

No offset `0x05`:

```
Offset 0x05:  01
```

- `01` = Little-Endian (LSB) — bytes menos significativos primeiro
- `02` = Big-Endian (MSB) — bytes mais significativos primeiro

ARM tipicamente usa Little-Endian.


### Exercício 4 — Identificar a arquitetura (e_machine)

Vá para o offset `0x12` com `F5` → `0x12`:

```
Offset 0x12:  b7 00
```

Como é Little-Endian, lemos como `0x00B7` = **183** = `EM_AARCH64`.


## Comparando ELF32 vs ELF64 no `ht` lado a lado

Abra os dois arquivos e navegue para o offset `0x04` em cada um:

```bash
# Terminal 1
ht hello_64

# Terminal 2 (novo terminal)
ht hello_32
```

```
hello_64 — Offset 0x00:
00000000  7f 45 4c 46 [02] 01 01 00  00 00 00 00 00 00 00 00
                       ^
                       02 = ELF64

hello_32 — Offset 0x00:
00000000  7f 45 4c 46 [01] 01 01 00  00 00 00 00 00 00 00 00
                       ^
                       01 = ELF32
```

E nos offsets de `e_machine` (0x12):

```
hello_64 — Offset 0x12:  b7 00  → 0x00B7 = 183 = EM_AARCH64
hello_32 — Offset 0x12:  28 00  → 0x0028 = 40  = EM_ARM
```

> **Você acabou de ler e interpretar um arquivo ELF manualmente, byte a byte!** Isso é o coração da Engenharia Reversa.


## Resumo visual de tudo que aprendemos

```
                    MAPA DO ELF HEADER (64-bit)
                    ============================

Offset  Bytes                    Significado
─────  ─────────────────────  ─────────────────────────────────────
0x00    7F 45 4C 46              Magic: "\x7FELF"
0x04    02                       Classe: ELF64
0x05    01                       Endianness: Little-Endian
0x06    01                       Versão ELF
0x07    00                       OS/ABI: System V
0x08    00                       EI_ABIVERSION: versão da ABI (0 = não especif.)
0x09    00 00 00 00 00 00 00     EI_PAD: padding (7 bytes zeros)
0x10    03 00                    e_type: ET_DYN (PIE executable)
0x12    B7 00                    e_machine: EM_AARCH64 (183)
0x14    01 00 00 00              e_version: 1
0x18    ?? ?? ?? ?? ?? ?? ?? ??  e_entry: endereço do entry point (8 bytes)
0x20    ?? ?? ?? ?? ?? ?? ?? ??  e_phoff: offset da Phdr Table (8 bytes)
0x28    ?? ?? ?? ?? ?? ?? ?? ??  e_shoff: offset da Shdr Table (8 bytes)
0x30    00 00 00 00              e_flags: flags da arquitetura
0x34    40 00                    e_ehsize: 64 bytes
0x36    38 00                    e_phentsize: 56 bytes
0x38    ?? 00                    e_phnum: qtd de Program Headers
0x3A    40 00                    e_shentsize: 64 bytes
0x3C    ?? 00                    e_shnum: qtd de Section Headers
0x3E    ?? 00                    e_shstrndx: índice da .shstrtab
        ↑ Total: 64 bytes de header
```


# Recapitulando...

## O que vimos até agora:

• O que é o formato ELF e para que serve  
• Diferenças entre ELF (Linux) e PE (Windows)  
• Como preparar um ambiente completo no Raspberry Pi 4  
• A diferença entre binários 32-bit e 64-bit no ARM  
• Como compilar programas C para ELF 64-bit e 32-bit  
• Como usar `file`, `readelf`, `hexdump`, `nm`, `strings`  
• A estrutura completa do ELF Header campo a campo  
• O significado do Magic Number `7F 45 4C 46`  
• Como usar o `ht` para análise hexadecimal interativa  
• Como ler campos em Little-Endian manualmente  


> **Uma última palavra motivadora**: Engenharia Reversa parece intimidadora no início porque você está lendo a "linguagem nativa" do computador (bytes brutos, offsets, endianness). Mas você já provou que consegue! Você leu um ELF Header byte a byte com suas próprias mãos. Cada vez que você abre um binário, você está vendo o mundo como o kernel vê. Continue explorando e bora lá pro volume 2!


*Guia elaborado para estudantes de Segurança e Engenharia Reversa — Formato ELF v1.0*  
*Ambiente: Raspberry Pi 4, ARM64 (AArch64), Raspberry Pi OS 64-bit*
Até lá!
