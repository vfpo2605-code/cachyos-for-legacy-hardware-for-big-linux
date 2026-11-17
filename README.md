# CachyOS BORE para BigLinux / Manjaro / Arch

Compilação do kernel **CachyOS BORE** a partir do `PKGBUILD` oficial do CachyOS, com perfil de compilação direcionado a **GCC**, **x86-64-v2**, **1000 Hz** e **LTO desativado**.

O script automatiza a preparação do ambiente, a obtenção do código-fonte, a adaptação pontual do `PKGBUILD`, a compilação com `makepkg` e, por padrão, a instalação dos pacotes gerados, a geração do initramfs e a atualização do GRUB.

> [!IMPORTANT]
> Este é um script de compilação e instalação de kernel. Um erro de configuração do boot pode impedir a inicialização do sistema. Leia as seções de **Pré-requisitos**, **Instalação e boot**, **Limitações** e **Recuperação** antes de executá-lo.

---

## Índice

- [Visão geral](#visão-geral)
- [Características](#características)
- [Perfil de compilação](#perfil-de-compilação)
- [Compatibilidade](#compatibilidade)
- [Pré-requisitos](#pré-requisitos)
- [Preparação](#preparação)
- [Uso](#uso)
- [O que o script faz](#o-que-o-script-faz)
- [Arquivos e diretórios gerados](#arquivos-e-diretórios-gerados)
- [Instalação e boot](#instalação-e-boot)
- [Secure Boot](#secure-boot)
- [DKMS e módulos externos](#dkms-e-módulos-externos)
- [Segurança e preservação do sistema](#segurança-e-preservação-do-sistema)
- [Solução de problemas](#solução-de-problemas)
- [Desinstalação e limpeza](#desinstalação-e-limpeza)
- [Limitações conhecidas](#limitações-conhecidas)
- [Informações para auditoria](#informações-para-auditoria)
- [Licença](#licença)

---

## Visão geral

O projeto compila o kernel CachyOS com o escalonador **BORE** usando o `PKGBUILD` mantido no repositório oficial:

- Repositório: <https://github.com/CachyOS/linux-cachyos>
- Diretório do pacote utilizado: `linux-cachyos-bore`

O script não contém uma cópia fixa do código-fonte do kernel. A cada execução, ele faz um clone raso (`--depth=1`) do repositório e utiliza a revisão disponível no `HEAD` naquele momento. O hash dessa revisão é registrado para permitir identificar a origem daquela compilação.

O nome `cachyos-v2-biglinux` é utilizado para organizar os diretórios de trabalho. O script menciona BigLinux, Manjaro e Arch como sistemas-alvo, mas não faz uma detecção rígida da distribuição nem garante compatibilidade com todas as versões ou configurações dessas distribuições.

## Características

| Recurso | Comportamento |
|---|---|
| Base da compilação | `PKGBUILD` oficial do CachyOS, pacote `linux-cachyos-bore` |
| Escalonador | BORE, solicitado por `_cpusched=bore` |
| Compilador | GCC para compilação do kernel e ferramentas host |
| Arquitetura | `x86_64` |
| Otimização de processador | Perfil `generic_v2` (x86-64-v2) |
| Frequência de tick | 1000 Hz |
| Tick rate | `full` |
| Preempção | `full` |
| Huge pages | Opção upstream `always` |
| LTO | Desativado pelas opções exportadas e pela configuração do `makepkg` |
| Paralelismo | Calculado com base na RAM disponível e limitado a 8 jobs |
| Empacotamento | `makepkg`, gerando pacotes instaláveis pelo Pacman |
| Initramfs | `mkinitcpio` ou, como alternativa, `dracut` |
| Bootloader | Atualização automática do GRUB quando uma configuração compatível é encontrada |
| Kernels anteriores | Não são removidos pelo script |
| Modo de compilação | `--build-only`, sem instalar os pacotes de kernel gerados |

## Perfil de compilação

O script exporta as seguintes opções para o `PKGBUILD` do CachyOS:

| Variável | Valor | Finalidade no script |
|---|---|---|
| `ARCH` | `x86_64` | Define a arquitetura de compilação |
| `CC` / `HOSTCC` | `gcc` | Solicita GCC como compilador do kernel e das ferramentas host |
| `LD` / `HOSTLD` | `ld.bfd` | Solicita o linker GNU BFD |
| `_cachy_config` | `yes` | Ativa a configuração CachyOS prevista pelo pacote upstream |
| `_cpusched` | `bore` | Seleciona o perfil BORE |
| `_processor_opt` | `generic_v2` | Solicita otimização para o nível x86-64-v2 |
| `_HZ_ticks` | `1000` | Solicita frequência de tick de 1000 Hz |
| `_use_llvm_lto` | `none` | Desativa a opção de LTO baseada em LLVM do pacote |
| `_use_lto_suffix` | `no` | Desativa o sufixo de pacote associado a LTO |
| `_use_gcc_suffix` | `yes` | Solicita o sufixo/identificação GCC previsto pelo upstream |
| `_tickrate` | `full` | Solicita tick rate completo |
| `_preempt` | `full` | Solicita preempção completa |
| `_hugepage` | `always` | Solicita a opção upstream de huge pages |
| `_localmodcfg` | `no` | Não solicita uma configuração limitada aos módulos atualmente carregados |
| `_use_current` | `no` | Não solicita a reutilização da configuração do kernel em execução |
| `_makenconfig` / `_makexconfig` | `no` | Não solicita os configuradores interativos indicados por essas opções |
| `_build_zfs` | `no` | Não solicita o pacote/módulo ZFS opcional do build upstream |
| `_build_nvidia_open` | `no` | Não solicita o módulo NVIDIA open opcional |
| `_build_nvidia_580xx` | `no` | Não solicita o módulo NVIDIA 580xx opcional |
| `_build_r8125` | `no` | Não solicita o módulo r8125 opcional |
| `_build_debug` | `no` | Não solicita os pacotes de debug opcionais |

Também é criada uma configuração própria do `makepkg` para cada execução. Ela carrega `/etc/makepkg.conf` e substitui alguns parâmetros:

- `CFLAGS` e `CXXFLAGS`: `-O2 -pipe`.
- `MAKEFLAGS`: `-j` seguido do número de jobs calculado pelo script.
- `BUILDENV`: `!distcc color !ccache check !sign`.
- `OPTIONS`: `!strip !debug !lto`.
- `CPPFLAGS` e `LDFLAGS`: vazios nessa configuração.

As variáveis de ambiente relacionadas a LLVM, compilação cruzada e flags externas de compilação são removidas antes do build, para reduzir a interferência de configurações herdadas do shell.

> [!NOTE]
> Essas opções são passadas ao `PKGBUILD` upstream. O resultado final depende de como a revisão do `PKGBUILD` obtida naquela execução interpreta essas variáveis. O script registra a revisão upstream e salva as versões original e modificada do arquivo para inspeção.

## Compatibilidade

### Distribuições

O script foi escrito para distribuições da família Arch Linux que utilizam:

- Pacman como gerenciador de pacotes;
- `makepkg` e `PKGBUILD` para compilar pacotes;
- estrutura de módulos em `/usr/lib/modules`;
- `/boot` como local para a imagem do kernel e os initramfs;
- GRUB, caso se deseje a atualização automática do menu de inicialização.

BigLinux, Manjaro e Arch são citados no cabeçalho do script. A presença dessas distribuições no comentário não representa uma garantia de funcionamento em todas as instalações. Sistemas com organização de boot diferente ou modificações específicas podem exigir etapas manuais.

### Processador

O perfil `generic_v2` tem como alvo o nível de recursos de CPU **x86-64-v2**. Portanto, o processador precisa oferecer esse conjunto de instruções. Um processador que suporte apenas o nível x86-64 básico pode não conseguir inicializar esse kernel.

O script imprime a arquitetura informada por `uname -m`, mas não realiza uma verificação completa das instruções suportadas pela CPU antes da compilação.

### Bootloader

A automação pós-instalação foi implementada para **GRUB**. O script não oferece uma rotina equivalente para systemd-boot, rEFInd ou outros gerenciadores de inicialização.

## Pré-requisitos

### Ferramentas exigidas antes da execução

O script verifica a presença destes comandos antes de instalar os pacotes auxiliares:

- `bash`;
- `git`;
- `makepkg`;
- `pacman`;
- `sudo`;
- `python3`;
- `gpg`.

Para executar o script com `sudo`, também é necessário `runuser`, pois ele é utilizado para retornar ao usuário original antes de chamar o `makepkg`.

### Pacotes auxiliares

Depois das verificações iniciais, o script solicita ao Pacman:

```bash
sudo pacman -S --needed --noconfirm \
  base-devel git gnupg libarchive zstd python
```

Esses pacotes fornecem ou complementam o ambiente de compilação e empacotamento. O comando utiliza `--needed` e não executa uma atualização geral do sistema.

### Ferramentas opcionais detectadas

| Ferramenta | Uso |
|---|---|
| `mokutil` | Consulta informativa do estado do Secure Boot |
| `dkms` | Compila módulos DKMS para o novo kernel, se estiver instalado |
| `mkinitcpio` | Gera initramfs; tem prioridade sobre `dracut` |
| `dracut` | Alternativa para gerar initramfs quando `mkinitcpio` não existe |
| `grub-mkconfig` | Regenera o arquivo de configuração do GRUB |
| `grub-set-default` | Tenta selecionar o novo kernel como entrada padrão |
| `grub-editenv` | Verifica a disponibilidade do mecanismo de entrada salva do GRUB |

A ausência de ferramentas opcionais pode fazer com que etapas posteriores sejam ignoradas ou concluídas apenas parcialmente, com um aviso no log.

### Recursos de hardware e espaço

A compilação do kernel pode consumir bastante memória, CPU, espaço em disco e tempo. O script não impõe um mínimo fixo de RAM ou armazenamento. Isso não significa que qualquer máquina tenha recursos suficientes para concluir a compilação.

Antes de iniciar, confira se há espaço livre para o clone do repositório, fontes, arquivos temporários, pacotes e logs. Mantenha também espaço adequado em `/boot` para o kernel e os initramfs.

## Preparação

1. Confirme que o sistema está inicializando normalmente com o kernel atual.
2. Faça backup dos dados importantes e mantenha disponível uma forma de iniciar por outro kernel ou mídia de recuperação.
3. Verifique se a partição ou o sistema de arquivos correto está montado em `/boot`.
4. Confirme que o sistema e os bancos de dados do Pacman estão em um estado consistente. O script não executa `pacman -Syu` nem atualiza os bancos com `pacman -Sy` isoladamente.
5. Confira o suporte x86-64-v2 do processador.
6. Leia as seções de Secure Boot e limitações antes de instalar o kernel.

O script grava os arquivos de boot diretamente em `/boot`. Ele não verifica se a partição esperada está montada nesse caminho. Executá-lo com `/boot` desmontado pode gravar arquivos no diretório subjacente, em vez da partição de boot real.

## Uso

Coloque o script em um diretório de sua preferência e torne-o executável:

```bash
chmod +x build-kernel-cachyos-v2-biglinux.sh
```

### Ajuda

```bash
./build-kernel-cachyos-v2-biglinux.sh --help
```

### Compilar e instalar

```bash
./build-kernel-cachyos-v2-biglinux.sh
```

Esse é o modo padrão. Depois de compilar, o script instala todos os pacotes gerados e tenta configurar initramfs e GRUB.

### Apenas compilar, sem instalar o kernel gerado

```bash
./build-kernel-cachyos-v2-biglinux.sh --build-only
```

O modo `--build-only` compila e preserva os pacotes locais, mas não executa a instalação dos pacotes de kernel, não copia a imagem do kernel para `/boot`, não gera initramfs e não atualiza o GRUB.

> [!NOTE]
> `--build-only` não significa que o sistema não sofrerá nenhuma alteração: o script ainda pode instalar ferramentas auxiliares e dependências de compilação por meio do Pacman.

### Execução com sudo

O modo recomendado é executar como usuário normal, permitindo que o script solicite privilégios via `sudo` quando necessário.

Também é aceito:

```bash
sudo bash ./build-kernel-cachyos-v2-biglinux.sh
```

Nesse caso, quando `SUDO_USER` identifica um usuário normal, o script usa `runuser` para reiniciar sua execução como esse usuário. Isso evita executar o `makepkg` como root.

Não execute a partir de uma sessão root sem `SUDO_USER` apontando para um usuário normal. Nessa situação, o script encerra com erro.

### Argumentos aceitos

| Argumento | Efeito |
|---|---|
| `--build-only` | Compila e mantém os pacotes sem instalá-los |
| `-h`, `--help` | Exibe a ajuda e encerra |
| Qualquer outro | Exibe erro de argumento desconhecido e encerra com código 2 |

## O que o script faz

A execução é organizada nas etapas abaixo.

### 1. Validação inicial e identidade do usuário

- Ativa `set -Eeuo pipefail` e `umask 022`.
- Processa os argumentos recebidos.
- Impede que `makepkg` seja executado como root.
- Se chamado por `sudo`, tenta voltar ao usuário original usando `runuser`.
- Verifica as ferramentas básicas necessárias.
- Solicita autenticação `sudo` antes de iniciar downloads e compilação.

### 2. Diagnóstico do sistema e cálculo de paralelismo

O script registra a distribuição identificada em `/etc/os-release`, a arquitetura, a versão do kernel em execução, a memória disponível, a quantidade de CPUs e, quando possível, o estado do Secure Boot.

O número de jobs é calculado a partir de aproximadamente **um job para cada 3 GiB de RAM disponível**, com estes limites:

- mínimo: 1 job;
- máximo: número de CPUs disponíveis;
- limite absoluto: 8 jobs;
- se a leitura da memória falhar, utiliza 2 jobs como valor inicial.

O cálculo é uma estimativa conservadora, não uma garantia contra falta de memória. Outros processos em execução também consomem recursos.

### 3. Instalação das ferramentas auxiliares

Executa `pacman -S --needed --noconfirm` para garantir os pacotes `base-devel`, `git`, `gnupg`, `libarchive`, `zstd` e `python`.

Não executa uma atualização completa do sistema e não usa `pacman -Sy` isoladamente.

### 4. Criação do ambiente privado de compilação

Cria uma pasta exclusiva para a execução, com data e hora no nome. Nela ficam o clone do repositório, os pacotes, as fontes, o diretório temporário do GnuPG, a configuração do `makepkg` e os logs.

O diretório `gnupg` recebe permissão `700`.

### 5. Clone do repositório e identificação da revisão

Clona o repositório oficial CachyOS com `--depth=1`, localiza `linux-cachyos-bore/PKGBUILD` e registra o hash do commit obtido em `pkgbuild-commit.txt`.

Uma cópia do `PKGBUILD` original é guardada antes de qualquer modificação.

### 6. Adaptação pontual do PKGBUILD

Um trecho Python procura padrões específicos no `PKGBUILD` e tenta aplicar duas adaptações:

1. inserir `make ... olddefconfig` antes da etapa `prepare` reconhecida, evitando o prompt de sincronização de configuração quando o padrão esperado é encontrado;
2. substituir a chamada de compilação reconhecida por uma chamada que define GCC e `ld.bfd` explicitamente e utiliza o número de jobs calculado.

As alterações são condicionais: dependem de o `PKGBUILD` upstream manter a estrutura e os comandos esperados pelas expressões regulares. O script salva uma cópia do arquivo modificado para que seja possível conferir o que realmente foi alterado.

O script não altera intencionalmente `pkgbase`, `_srcname`, fontes ou checksums do `PKGBUILD`.

### 7. Preparação das chaves de assinatura

O script extrai as chaves PGP de 40 caracteres listadas em `validpgpkeys` no `PKGBUILD` e tenta importá-las de:

```text
hkps://keyserver.ubuntu.com
```

Se a importação falhar, emite um aviso. A validação de assinatura e integridade realizada pelo `makepkg` continua sendo o mecanismo de validação do pacote; uma falha nessa validação pode interromper a compilação.

O GnuPG utiliza um `GNUPGHOME` temporário dentro do diretório daquela execução.

### 8. Geração de metadados e compilação

Executa `makepkg --printsrcinfo` para gerar `.SRCINFO` e descobrir o `pkgbase` declarado pelo pacote upstream.

Em seguida, compila e empacota com:

```bash
makepkg --syncdeps --noconfirm --log
```

`--syncdeps` permite resolver dependências de compilação pelo gerenciador de pacotes. A compilação é feita como usuário normal, e não como root.

### 9. Localização dos pacotes gerados

Procura recursivamente no diretório de trabalho os arquivos compatíveis com os formatos de pacote do Pacman (`*.pkg.tar` e `*.pkg.tar.*`), ignorando arquivos dentro de diretórios `pkg`.

Se nenhum pacote for encontrado, o script imprime informações de diagnóstico e encerra com erro, preservando o diretório de build.

### 10. Modo de compilação sem instalação

Se `--build-only` foi informado, o script exibe os caminhos dos pacotes e do log, confirma a conclusão e encerra antes de instalar o kernel.

### 11. Instalação dos pacotes

No modo padrão, executa `pacman -U --noconfirm` com todos os pacotes localizados. Isso inclui os pacotes principais e eventuais split packages produzidos pelo `PKGBUILD`.

### 12. Preparação dos módulos e do initramfs

Depois da instalação, tenta identificar a versão (`kernelrelease`) do kernel a partir da árvore de fontes. Se isso não funcionar, procura em `/usr/lib/modules/*/pkgbase` um pacote com o `pkgbase` correspondente.

Quando encontra o diretório de módulos:

- executa `depmod -a` para a versão instalada;
- executa `dkms autoinstall -k` se `dkms` estiver disponível;
- copia `/usr/lib/modules/<kernelrelease>/vmlinuz` para `/boot/vmlinuz-<pkgbase>`;
- gera o initramfs padrão e o fallback, usando `mkinitcpio` ou `dracut`, conforme a disponibilidade.

Falhas em `depmod`, DKMS ou na geração do initramfs são registradas como avisos em determinadas etapas, e o script pode continuar. Por isso, uma mensagem final de conclusão não substitui a conferência dos logs e dos arquivos de boot.

### 13. Atualização do GRUB

Se `grub-mkconfig` estiver disponível, procura uma configuração existente em:

- `/boot/grub/grub.cfg`;
- `/boot/grub2/grub.cfg`.

Quando encontra uma delas, executa `grub-mkconfig`. Depois, tenta localizar no menu uma entrada que aponte para `vmlinuz-<pkgbase>`.

Se localizar a entrada e as ferramentas necessárias estiverem disponíveis, o script:

1. faz uma cópia de `/etc/default/grub` com sufixo de backup;
2. define `GRUB_DEFAULT=saved` em `/etc/default/grub`;
3. regenera a configuração do GRUB;
4. executa `grub-set-default` para selecionar a entrada CachyOS encontrada.

Se não encontrar a entrada ou a configuração do GRUB, emite um aviso e deixa a seleção para o usuário. Não executa `grub-install`.

### 14. Resumo e preservação do build

Ao final, mostra o `pkgbase`, a versão do kernel identificada, os diretórios de build e pacotes, o caminho do log e os kernels encontrados em `/usr/lib/modules`.

O diretório da compilação não é removido automaticamente, mesmo em caso de falha.

## Arquivos e diretórios gerados

Cada execução cria uma pasta com o padrão:

```text
~/cachyos-v2-biglinux-AAAAmmdd-HHMMSS/
```

Exemplo ilustrativo:

```text
~/cachyos-v2-biglinux-20261116-231500/
├── repo/
│   └── linux-cachyos-bore/
│       ├── PKGBUILD
│       ├── .SRCINFO
│       └── src/                 # fontes e árvore de compilação, conforme o makepkg
├── packages/                    # destino configurado para os pacotes
├── sources/                     # cache de fontes configurado para o makepkg
├── gnupg/                       # chaveiro temporário da execução
├── makepkg.conf                 # configuração isolada do makepkg
├── build.log                    # log geral da execução
├── pkgbuild-commit.txt          # hash do commit upstream utilizado
├── PKGBUILD.upstream             # cópia original do PKGBUILD
├── PKGBUILD.modified             # cópia após as adaptações do script
├── build-options.txt             # opções exportadas e pkgbase
└── packages-built.txt            # caminhos dos pacotes encontrados
```

A estrutura interna de `repo/` pode variar conforme o `PKGBUILD` upstream e o comportamento do `makepkg`. O script também procura os pacotes recursivamente dentro do diretório da execução, portanto a localização efetiva pode ser confirmada em `packages-built.txt`.

### Arquivos de boot instalados

No modo padrão, os nomes usados pelo script são:

```text
/boot/vmlinuz-<pkgbase>
/boot/initramfs-<pkgbase>.img
/boot/initramfs-<pkgbase>-fallback.img
```

O nome exato depende do `pkgbase` identificado no `PKGBUILD` upstream.

## Instalação e boot

Antes de reiniciar, confira manualmente:

```bash
# Verifique os diretórios de módulos instalados
ls -lah /usr/lib/modules

# Confira os arquivos de boot gerados
ls -lh /boot/vmlinuz-* /boot/initramfs-*.img 2>/dev/null

# Consulte o status dos módulos DKMS, se o DKMS estiver instalado
dkms status

# Confira a configuração do GRUB, se aplicável
grep -E '^(GRUB_DEFAULT|GRUB_TIMEOUT)=' /etc/default/grub
```

Os comandos de verificação são exemplos. A ausência de algum padrão de arquivo pode ser normal em sistemas cuja configuração de boot seja diferente; compare os resultados com o log da execução.

**Não reinicie se o kernel, o initramfs ou a entrada de boot esperada estiverem ausentes ou se o log mostrar uma falha relevante.** Mantenha o kernel anterior disponível no menu do GRUB até confirmar que o novo kernel inicializa e que os dispositivos essenciais funcionam.

Após iniciar pelo novo kernel, confirme:

```bash
uname -r
```

O resultado deve corresponder à versão instalada do kernel CachyOS. Teste também rede, armazenamento, áudio, vídeo e demais dispositivos importantes para sua máquina.

## Secure Boot

O script apenas tenta consultar o estado do Secure Boot por meio de `mokutil --sb-state`, quando `mokutil` está instalado.

Ele **não**:

- desativa o Secure Boot no firmware;
- assina o kernel ou os módulos;
- cria ou instala certificados;
- cadastra chaves MOK;
- configura uma cadeia de inicialização confiável.

Com Secure Boot ativo, um kernel personalizado sem assinatura reconhecida pela cadeia de confiança pode ser recusado durante a inicialização. A preparação de assinatura precisa ser feita separadamente, de acordo com a configuração do equipamento e da distribuição.

## DKMS e módulos externos

Se o comando `dkms` estiver disponível, o script tenta compilar os módulos DKMS existentes para a nova versão do kernel:

```bash
sudo dkms autoinstall -k <kernelrelease>
```

A falha de um ou mais módulos DKMS gera um aviso, mas não necessariamente interrompe todo o processo. Isso pode afetar drivers externos, como módulos de Wi-Fi, vídeo ou outros dispositivos instalados via DKMS.

Depois da compilação, confira:

```bash
dkms status
```

Não presuma que todos os módulos externos foram compilados com sucesso apenas porque os pacotes do kernel foram instalados.

## Segurança e preservação do sistema

O comportamento implementado pelo script inclui estas proteções e escolhas:

- **Não executa `makepkg` como root.** Se iniciado via `sudo`, tenta reexecutar a si mesmo como o usuário original.
- **Não remove kernels existentes.** Não há comandos de remoção de kernels anteriores.
- **Não executa atualização geral do sistema.** Não chama `pacman -Syu` nem `pacman -Sy` isoladamente.
- **Não limpa o diretório de compilação.** Logs, fontes e pacotes são preservados para inspeção.
- **Registra a revisão upstream.** O hash do commit utilizado fica em `pkgbuild-commit.txt`.
- **Preserva o PKGBUILD original.** A cópia `PKGBUILD.upstream` permite comparar a versão recebida com `PKGBUILD.modified`.
- **Usa um chaveiro GPG por execução.** O `GNUPGHOME` temporário fica dentro do diretório de build e recebe permissão restrita.
- **Não força limites arbitrários de RAM ou disco.** Faz apenas o cálculo do paralelismo e registra o diagnóstico.

> [!WARNING]
> Preservar os kernels existentes não elimina todos os riscos. A instalação de pacotes, a gravação em `/boot`, a geração de initramfs e a alteração de `GRUB_DEFAULT` são operações reais no sistema. Mantenha backups e um método alternativo de inicialização.

## Solução de problemas

### O script informa que uma ferramenta básica está ausente

A mensagem identifica o comando que não foi encontrado. Instale a ferramenta correspondente antes de executar novamente. As verificações de `git`, `makepkg`, `pacman`, `sudo`, `python3` e `gpg` acontecem antes da instalação dos pacotes auxiliares pelo próprio script.

### O script foi executado diretamente como root e encerrou

Execute como usuário normal:

```bash
./build-kernel-cachyos-v2-biglinux.sh
```

Ou use `sudo` a partir da conta normal:

```bash
sudo bash ./build-kernel-cachyos-v2-biglinux.sh
```

A segunda forma depende de `SUDO_USER` e `runuser` estarem disponíveis.

### A importação de uma chave GPG falhou

O script tenta buscar as chaves no keyserver Ubuntu. Uma falha de conexão gera um aviso, mas a verificação de assinatura do `makepkg` não é desativada. Confira a conectividade e o log. Não ignore uma falha de validação de assinatura para forçar a instalação.

### O PKGBUILD upstream mudou

As adaptações são feitas por busca de padrões no arquivo. Se o upstream mudar a estrutura de `prepare()` ou `build()`, uma ou ambas as alterações podem deixar de ser aplicadas.

Compare:

```text
PKGBUILD.upstream
PKGBUILD.modified
```

Confira também `build-options.txt` e `pkgbuild-commit.txt`. Se a estrutura upstream mudou, revise manualmente o `PKGBUILD.modified` antes de confiar no perfil de compilação esperado.

### A compilação falhou

Abra o log da execução:

```bash
less ~/cachyos-v2-biglinux-*/build.log
```

Para uma pasta específica, substitua o caminho pelo diretório mais recente. O script preserva o diretório de build e informa seu caminho quando um comando falha.

Consulte também:

- `pkgbuild-commit.txt`, para identificar a revisão upstream;
- `PKGBUILD.upstream` e `PKGBUILD.modified`, para inspecionar as adaptações;
- `build-options.txt`, para conferir as opções exportadas;
- `packages-built.txt`, se os pacotes chegaram a ser produzidos.

### Nenhum pacote `.pkg.tar*` foi encontrado

O script procura os pacotes recursivamente no diretório da execução. Se não localizar nenhum, imprime um diagnóstico dos diretórios de saída e encerra com erro. Verifique o log do `makepkg` para determinar se a compilação falhou ou se o formato/local de saída mudou.

### O kernel foi instalado, mas não há initramfs

O script tenta usar `mkinitcpio`; se o comando não existir, tenta `dracut`. Se nenhum dos dois estiver instalado, apenas emite um aviso. Instale e configure a ferramenta de initramfs apropriada para a distribuição e gere as imagens conforme a configuração local antes de reiniciar.

### Um módulo DKMS falhou

Consulte:

```bash
dkms status
```

Verifique os logs específicos do módulo e confirme se ele oferece suporte à versão do kernel compilado. O script não consegue garantir compatibilidade de módulos externos.

### O GRUB não mostra a nova entrada

Confira se `grub-mkconfig` está instalado, se existe um dos caminhos de configuração reconhecidos pelo script e se a imagem `/boot/vmlinuz-<pkgbase>` foi criada.

O script não executa `grub-install` e não cobre todos os layouts de GRUB. Em caso de falha, regenere a configuração de acordo com a documentação da distribuição e selecione o kernel manualmente. Não remova o kernel anterior.

### O novo kernel não inicia

Use a entrada do kernel anterior no menu de boot ou uma mídia de recuperação. Verifique se:

- a CPU suporta x86-64-v2;
- o Secure Boot permite inicializar o kernel;
- a imagem e o initramfs correspondem à mesma instalação;
- os módulos DKMS necessários foram compilados;
- a partição correta estava montada em `/boot` durante a execução.

O script não possui uma rotina automática de rollback.

## Desinstalação e limpeza

O script **não implementa desinstalação**. Ele também não apaga os diretórios de build.

Para remover o kernel posteriormente, identifique primeiro os nomes exatos dos pacotes instalados e consulte o gerenciador de pacotes da distribuição. Não copie um comando genérico de remoção sem conferir as dependências e sem garantir que outro kernel inicializável continuará instalado.

Os diretórios de compilação podem ser removidos manualmente depois que você não precisar mais dos logs, fontes ou pacotes. Antes de apagar, confira se os pacotes ainda são necessários para reinstalação ou diagnóstico.

O script pode criar um backup de configuração do GRUB com este padrão:

```text
/etc/default/grub.before-cachyos-v2-biglinux-AAAAmmdd-HHMMSS
```

Ele não restaura esse arquivo automaticamente. A restauração, se necessária, deve ser feita manualmente após comparar o backup com a configuração atual.

## Limitações conhecidas

1. **Compatibilidade depende do upstream.** O script clona o `HEAD` atual do repositório CachyOS, sem fixar uma tag ou commit predefinido.
2. **As adaptações do PKGBUILD dependem de padrões textuais.** Mudanças na estrutura upstream podem impedir que `olddefconfig` ou a chamada GCC sejam inseridos.
3. **O suporte automático de boot é focado em GRUB.** Não há configuração equivalente para systemd-boot, rEFInd ou outros bootloaders.
4. **Não verifica se `/boot` está montado corretamente.** Essa conferência precisa ser feita pelo usuário antes da execução.
5. **Não assina o kernel para Secure Boot.** A assinatura e o cadastro de chaves são externos ao script.
6. **Não garante sucesso de todos os módulos DKMS.** A compilação é tentada, e falhas são reportadas como avisos.
7. **Pode continuar depois de algumas falhas pós-instalação.** A geração de initramfs, `depmod`, DKMS e atualização do GRUB possuem caminhos de aviso; revise o log antes de reiniciar.
8. **Não faz rollback automático.** O kernel anterior é preservado, mas alterações feitas durante a execução não são revertidas automaticamente.
9. **Não instala dependências que faltem antes das verificações iniciais.** Os comandos básicos listados em Pré-requisitos precisam existir quando o script começa.
10. **Não garante recursos de hardware.** O cálculo de jobs limita o paralelismo, mas não verifica se há RAM ou espaço em disco suficientes para concluir o build.
11. **Não atualiza o kernel automaticamente no futuro.** Uma nova versão exige uma nova execução do script.
12. **Não declara uma política de limpeza.** Os diretórios de build permanecem no disco até serem removidos manualmente.

## Informações para auditoria

Para reproduzir ou investigar uma compilação, mantenha juntos estes arquivos da mesma execução:

| Arquivo | O que permite verificar |
|---|---|
| `pkgbuild-commit.txt` | Revisão exata do repositório CachyOS clonada |
| `PKGBUILD.upstream` | PKGBUILD recebido do upstream antes das alterações |
| `PKGBUILD.modified` | PKGBUILD efetivamente preparado pelo script |
| `build-options.txt` | Opções exportadas e `pkgbase` identificado |
| `makepkg.conf` | Configuração específica usada pelo `makepkg` |
| `build.log` | Saída de diagnóstico, compilação, instalação e pós-instalação |
| `packages-built.txt` | Caminhos dos pacotes produzidos e localizados |

O conjunto acima permite conferir a revisão e as modificações do pacote, mas não torna o build completamente reproduzível por si só: fontes externas, dependências, ambiente do sistema e conteúdo upstream podem mudar.

## Licença

O script não declara uma licença de distribuição em seu conteúdo. Antes de publicar ou redistribuir o projeto, inclua um arquivo `LICENSE` com a licença que o mantenedor efetivamente escolher e tiver direito de aplicar.

---

<p align="center">
  <sub>Documentação do script <code>build-kernel-cachyos-v2-biglinux.sh</code>.</sub>
</p>
