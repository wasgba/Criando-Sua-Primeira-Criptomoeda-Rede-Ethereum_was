# Scripts para Automação da Criação e Configuração de uma Rede Privada Ethereum

Este documento fornece scripts e instruções para automatizar a criação e configuração de uma rede blockchain privada utilizando a plataforma Ethereum. Os scripts são compatíveis com Linux, MacOS e Windows (via WSL).

## 1 - Script para o Nó de Inicialização (Bootnode): `boot.sh`

Salve o conteúdo abaixo como `boot.sh` e torne-o executável (`chmod +x boot.sh`).

```bash
#!/bin/bash
#
# boot.sh – Script de inicialização do Bootnode para rede privada Ethereum
#
# Este script gera (ou utiliza) uma chave para o bootnode, garante que a
# pasta dos dados exista e exibe na tela a URL de conexão (enode)
#
# Parâmetros ajustáveis:
#    -v Versão do cliente Ethereum (se necessário)
#    -n NetworkID (deve coincidir com o chainId do genesis.json)
#    -d Diretório dos dados (padrão: $HOME/.ethereum/private/boot)
#    -k Chave do bootnode (arquivo de chave; se não existir, será gerada)
#    -b IP do bootnode (padrão: 127.0.0.1)
#    -p Porta do bootnode (padrão: 30301)

# Configurações padrão
VERSAO="latest"
NETWORKID="1288"
BOOTNODEDATADIR="$HOME/.ethereum/private/boot"
BOOTNODEKEY="bootnode.key"
BOOTNODEIP="127.0.0.1"
BOOTNODEPORT="30301"

usage_boot() {
  echo "Uso: $0 [-v Versão] [-n NetworkID] [-d BootnodeDataDir] [-k BootnodeKey] [-b BootnodeIP] [-p BootnodePort]"
  exit 1
}

# Processar os parâmetros
while getopts "v:n:d:k:b:p:" OPTION; do
  case "$OPTION" in
    v) VERSAO="$OPTARG" ;;
    n) NETWORKID="$OPTARG" ;;
    d) BOOTNODEDATADIR="$OPTARG" ;;
    k) BOOTNODEKEY="$OPTARG" ;;
    b) BOOTNODEIP="$OPTARG" ;;
    p) BOOTNODEPORT="$OPTARG" ;;
    *) usage_boot ;;
  esac
done

# Criar diretório de dados, se não existir
mkdir -p "$BOOTNODEDATADIR"
cd "$BOOTNODEDATADIR" || { echo "Erro ao acessar $BOOTNODEDATADIR"; exit 1; }

# Se a chave do bootnode não existir, gerá-la
if [ ! -f "$BOOTNODEKEY" ]; then
  echo "Chave $BOOTNODEKEY não encontrada, gerando nova chave..."
  bootnode -genkey "$BOOTNODEKEY" || { echo "Erro ao gerar a chave do bootnode."; exit 1; }
fi

# Gerar e exibir a URL de conexão (enode)
# O comando abaixo utiliza bootnode em modo verboroso para extrair o enode
BOOTNODE_ENODE=$(bootnode -nodekey "$BOOTNODEKEY" -verbosity 3 2>&1 | grep -o 'enode://[^ ]*')
if [ -z "$BOOTNODE_ENODE" ]; then
  echo "Falha ao gerar o enode."
  exit 1
fi

echo "Bootnode iniciado com sucesso!"
echo "Enode: $BOOTNODE_ENODE"

# 2 - Script para Iniciar e Parar os Nós (Aplicação ou Minerador):
start.sh
Este script permite iniciar ou parar um nó do tipo "node" (aplicação) ou "mine" (minerador), com opções para definir a porta, diretório de dados, referência para o bootnode, etc. Salve-o como start.sh e também torne-o executável.



#!/bin/bash
#
# start.sh – Script para iniciar/encerrar nós de rede privada Ethereum
#
# Tipos de nó:
#    - node   : Nó de aplicação com servidor HTTP-RPC ativado.
#    - mine   : Nó minerador (para mineração de blocos).
#
# Operações:
#    - start  : Inicia o nó (padrão).
#    - stop   : Para o nó (procura o processo geth a partir do diretório de dados).
#
# Uso:
#    ./start.sh -t <node|mine> [-o <start|stop>] [-p <MYNODEPORT>] [-d <DATADIR>]
#              [-i <BOOTNODEIP>] [-b <BOOTNODEID>] [-r <BOOTNODEPORT>] [-n <NETWORKID>]
#
# Observação: O parâmetro -b (BOOTNODEID, ou seja, o “enode” do bootnode) é obrigatório.

# Configurações padrão
NODETYPE="node"        # 'node' para aplicação ou 'mine' para minerador
OPERATIONTYPE="start"    # start (padrão) ou stop
MYNODEPORT="30303"
DATADIR="$HOME/.ethereum/private/node"
BOOTNODEIP="127.0.0.1"
BOOTNODEID=""            # (Obrigatório) Enode do bootnode, sem o prefixo "enode://"
BOOTNODEPORT="30301"
NETWORKID="1288"

usage_start() {
  cat <<EOL
Uso: $0 -t <node|mine> [-o <start|stop>] [-p <MYNODEPORT>] [-d <DATADIR>] [-i <BOOTNODEIP>] [-b <BOOTNODEID>] [-r <BOOTNODEPORT>] [-n <NETWORKID>]

    -t : Tipo de nó. 'node' para aplicação (HTTP-RPC) ou 'mine' para minerador.
    -o : Operação: 'start' (padrão) para iniciar ou 'stop' para finalizar o nó.
    -p : Porta do nó local (padrão 30303; se for minerador pode ser alterada, ex: 30304).
    -d : Diretório de dados onde os arquivos da rede são armazenados (padrão: \$HOME/.ethereum/private/node).
    -i : IP do bootnode (padrão: 127.0.0.1).
    -b : ID do bootnode (enode) – O parâmetro é obrigatório.
    -r : Porta do bootnode (padrão: 30301).
    -n : NetworkID (chainId) (padrão: 1288).

Exemplo para iniciar um nó de aplicação:
    ./start.sh -t node -b abcdef...123456

Exemplo para iniciar um nó minerador com porta diferenciada:
    ./start.sh -t mine -p 30304 -b abcdef...123456
EOL
  exit 1
}

# Processar os parâmetros
while getopts "t:o:p:d:i:b:r:n:" OPTION; do
  case "$OPTION" in
    t) NODETYPE="$OPTARG" ;;
    o) OPERATIONTYPE="$OPTARG" ;;
    p) MYNODEPORT="$OPTARG" ;;
    d) DATADIR="$OPTARG" ;;
    i) BOOTNODEIP="$OPTARG" ;;
    b) BOOTNODEID="$OPTARG" ;;
    r) BOOTNODEPORT="$OPTARG" ;;
    n) NETWORKID="$OPTARG" ;;
    *) usage_start ;;
  esac
done

if [ -z "$BOOTNODEID" ]; then
  echo "Erro: BOOTNODEID (enode) deve ser especificado com a flag -b."
  usage_start
fi

# Função para iniciar/parar o nó
start_node() {
  echo "Tipo de nó: $NODETYPE"
  echo "Operação: $OPERATIONTYPE"
  echo "Diretório de dados: $DATADIR"
  mkdir -p "$DATADIR"

  # Monta o comando base do geth
  CMD="geth --datadir $DATADIR --networkid $NETWORKID --port $MYNODEPORT --bootnodes enode://$BOOTNODEID@$BOOTNODEIP:$BOOTNODEPORT"

  if [ "$NODETYPE" = "mine" ]; then
    # Para nó minerador, ativa a mineração com 1 thread
    CMD="$CMD --mine --miner.threads=1"
  elif [ "$NODETYPE" = "node" ]; then
    # Para nó de aplicação, ativa a API HTTP (ajuste as opções conforme necessário)
    CMD="$CMD --http --http.addr 127.0.0.1 --http.port 8545 --http.api eth,net,web3,personal --http.corsdomain '*'"
  else
    echo "Tipo de nó '$NODETYPE' não reconhecido. Utilize 'node' ou 'mine'."
    usage_start
  fi

  if [ "$OPERATIONTYPE" = "start" ]; then
    echo "Executando: $CMD"
    # Executa o nó em background (nohup garante que continue funcionando)
    nohup $CMD >> "$DATADIR/node.log" 2>&1 &
    echo "Nó iniciado. Verifique os logs em: $DATADIR/node.log"
  elif [ "$OPERATIONTYPE" = "stop" ]; then
    # Para encerrar o nó, procura o processo geth que utiliza o diretório de dados especificado
    PID=$(pgrep -f "geth --datadir $DATADIR")
    if [ -z "$PID" ]; then
      echo "Nenhum processo geth encontrado para o diretório $DATADIR."
    else
      kill $PID
      echo "Processo geth (PID $PID) finalizado."
    fi
  else
    echo "Operação '$OPERATIONTYPE' não reconhecida. Utilize 'start' ou 'stop'."
    usage_start
  fi
}

start_node

# 3 - Ajustando o Arquivo genesis.json
O arquivo genesis.json define as propriedades iniciais da sua rede blockchain. Edite-o conforme suas necessidades, como alterar o chainId, a dificuldade de mineração ou os endereços pré-financiados.


{
  "config": {
    "chainId": 1288,                        // Identificador da rede
    "homesteadBlock": 0,
    "eip150Block": 0,
    "eip150Hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "eip155Block": 0,
    "eip158Block": 0,
    "byzantiumBlock": 0,
    "constantinopleBlock": 0,
    "petersburgBlock": 0,
    "istanbulBlock": 0,
    "ethash": {}
  },
  "nonce": "0x0",
  "timestamp": "0x5f527daa",
  "extraData": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "gasLimit": "0x2fefd8ffffffffff",
  "difficulty": "0x80000",                   // Dificuldade baixa para facilitar testes e desenvolvimento
  "mixHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "coinbase": "0x0000000000000000000000000000000000000000",
  "alloc": {
    "40ebd26453a3a3ec06df9b1cf6cb17355d95e78d": {
      "balance": "0x200000000000000000000000000000000000000000000000000000000000000"
    },
    "fd96fcc76da5e04604270bac93cd0e2acdcd670d": {
      "balance": "0x200000000000000000000000000000000000000000000000000000000000000"
    }
  },
  "number": "0x0",
  "gasUsed": "0x0",
  "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000"
}

# 4 - Funcionamento Geral e Instruções de Uso

Configurar genesis.json:  Edite o arquivo genesis.json para definir as propriedades iniciais da rede e pré-financiar as contas.


Iniciar o Bootnode:
./boot.sh -b [IP_desejado] -p [Porta_desejada]  # Ou apenas ./boot.sh para os padrões
Anote o enode gerado pelo bootnode, pois será necessário para os outros nós.


Iniciar um Nó de Aplicação:


./start.sh -t node -b [BOOTNODE_ENODE_sem_prefixo]


Este script iniciará o geth com a API HTTP ativada.


Iniciar um Nó Minerador:


./start.sh -t mine -p 30304 -b [BOOTNODE_ENODE_sem_prefixo] # Se estiver na mesma máquina, altere a porta


Parar um Nó:


./start.sh -t node -o stop -d [diretório_do_nó]
# ou
./start.sh -t mine -o stop -d [diretório_do_nó]


Observações Finais

Se estiver utilizando o Windows, considere utilizar o Subsistema do Windows para Linux (WSL).

Certifique-se de que os binários (geth, bootnode) estejam instalados e no PATH do sistema.

Armazene senhas (senha.txt, .accountpassword) e chaves privadas (privado.txt) de forma segura.
