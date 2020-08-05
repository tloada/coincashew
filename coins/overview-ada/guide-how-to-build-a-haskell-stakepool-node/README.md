
---
**CRÉDITOS: >-
  ESTA GUÍA FUE HECHA POR [COINCASHEW](https://www.coincashew.com).
  LA GUÍA ORIGINAL EN INGLÉS LA PUEDES ENCONTRAR** [**AQUÍ**](https://www.coincashew.com/coins/overview-ada/guide-how-to-build-a-haskell-stakepool-node/#15-operational-and-maintenance-tips).
---
---
TRADUCIDA POR: >-
  ESTA GUÍA FUE TRADUCIDA POR [THE LEGEND OF ₳DA POOL [TLOA]](https://tloada.github.io/tloa/español.html).
  SI DESEAS APOYARNOS, PUEDES HACERLO DELEGANDO A NUESTRO POOL CON TICKER [[TLOA]](https://tloada.github.io/tloa/español.html).
---
---
Descripción: >-
  En Ubuntu/Debian, esta guía ilustrará cómo instalar y configurar un
  stake pool de Cardano usando el código fuente.
---

# Guía: Cómo construir un Stake Pool de Cardano


A partir del 28 de julio, 2020, esta guía está escrita para **mainnet** con **edición v.1.18.0** 😁 

## 🏁 0. Prerequisitos

### 🧙♂ Habilidades de los operadores de stake pool

Como un operador de stake pool de Cardano, típicamente tendrás las siguientes habilidades:

* conocimiento operacional de cómo instalar, operar y mantener un nodo de Cardano continuamente
* un compromiso a mantenera tu nodo 24/7/365
* habilidades de sistemas operativos
* habilidades de administración de servidores \(operacionales y mantenimiento\)
* experiencia de desarrollo y operaciones \(DevOps\) sería muy útil

### 🎗 Requerimientos Mínimos del Equipo

* **Sistema Operativo:** 64-bit Linux \(i.e. Ubuntu 20.04 LTS\)
* **Procesador:** CPU con doble núcleo
* **Memoria RAM:** 4GB
* **Disco Duro:** 24GB
* **Internet:** conexión 24/7 a internet con banda ancha con velocidades de al menos 1 Mbps.
* **Plan de Datos**: como mínimo 1GB por hora. 720GB cada mes.
* **Electricidad:** energía eléctrica 24/7
* **Saldo de ADA:** como mínimo 1000 ADA

### 🏋♂ Equipo Recomendado para Largo Plazo

* **Sistema Operativo:** 64-bit Linux \(i.e. Ubuntu 20.04 LTS\)
* **Procesador:** CPU con cuádruple núcleo o mejor
* **Memoria RAM:** 16GB
* **Disco Duro:** 500GB SSD con RAID
* **Internet:** Múltiples conexiones 24/7 a internet con banda ancha con velocidades de al menos 10 Mbps \(i.e. fibra + celular 4G\)
* **Plan de Datos**: como mínimo 1GB por hora. 720GB cada mes.
* **Electricidad:** energía eléctrica redundante 24/7 con SAI
* **Saldo de ADA:** más pledge es mejor, será determinado por **a0**, el factor que influye al pledge

Nota que la velocidad del pprocesador no es un factor determinante para dirigir un stake pool.

Si estás reconstruyendo o reusando una instalción existente de `cardano-node`, [refiérete a la sección 15.2 de cómo resetear la instalación](./#15-2-resetting-the-installation)

## 🏭 1. Instala Cabal y GHC

**Oprime** Ctrl+Alt+T. Esto lanzará la terminal en una ventana. 

Primeramente, actualiza los paquetes e instala las dependencias de Ubuntu.

```text
sudo apt-get update -y
sudo apt-get upgrade -y
sudo apt-get install git make tmux rsync htop curl build-essential pkg-config libffi-dev libgmp-dev libssl-dev libtinfo-dev libsystemd-dev zlib1g-dev make g++ tmux git jq wget libncursesw5 libtool autoconf -y
```

Instala Libsodium.

```text
mkdir ~/git
cd ~/git
git clone https://github.com/input-output-hk/libsodium
cd libsodium
git checkout 66f017f1
./autogen.sh
./configure
make
sudo make install
```

Instala Cabal.

```text
cd
wget https://downloads.haskell.org/~cabal/cabal-install-3.2.0.0/cabal-install-3.2.0.0-x86_64-unknown-linux.tar.xz
tar -xf cabal-install-3.2.0.0-x86_64-unknown-linux.tar.xz
rm cabal-install-3.2.0.0-x86_64-unknown-linux.tar.xz cabal.sig
mkdir -p ~/.local/bin
mv cabal ~/.local/bin/
```

Instala GHC.

```text
wget https://downloads.haskell.org/~ghc/8.6.5/ghc-8.6.5-x86_64-deb9-linux.tar.xz
tar -xf ghc-8.6.5-x86_64-deb9-linux.tar.xz
rm ghc-8.6.5-x86_64-deb9-linux.tar.xz
cd ghc-8.6.5
./configure
sudo make install
```

Actualiza el PATH para que incluya Cabal y GHC y agrega los exportes. La dirección a tu nodo será **$NODE\_HOME**. La [agrupación de la configuración](https://hydra.iohk.io/job/Cardano/iohk-nix/cardano-deployment/latest-finished/download/1/index.html) es establecida por **$NODE\_CONFIG, $NODE\_URL** y **$NODE\_BUILD\_NUM**. 

```text
echo PATH="~/.local/bin:$PATH" >> ~/.bashrc
echo export LD_LIBRARY_PATH="/usr/local/lib:$LD_LIBRARY_PATH" >> ~/.bashrc
echo export NODE_HOME=$HOME/cardano-my-node >> ~/.bashrc
echo export NODE_CONFIG=mainnet>> ~/.bashrc
echo export NODE_URL=cardano-mainnet >> ~/.bashrc
echo export NODE_BUILD_NUM=$(curl https://hydra.iohk.io/job/Cardano/iohk-nix/cardano-deployment/latest-finished/download/1/index.html | grep -e "build" | sed 's/.*build\/\([0-9]*\)\/download.*/\1/g') >> ~/.bashrc
echo export NETWORK_IDENTIFIER=\"--mainnet\" >> ~/.bashrc
source ~/.bashrc
```

Actualiza cabal y verifica que las versiones correctas fueron instaladas correctamente.

```text
cabal update
cabal -V
ghc -V
```

La versión de la librería de Cabal debería de ser 3.2.0.0 y la versión de GHC debería de ser 8.6.5

## 🏗 2. Construyendo el nodo desde el código fuente

Descarga el código fuente y cambia al *tag* más reciente. En este caso usa `tags/1.18.0`

```text
cd ~/git
git clone https://github.com/input-output-hk/cardano-node.git
cd cardano-node
git fetch --all
git checkout tags/1.18.0
```

Actualiza cabal config, configuración del proyecto y resetea la carpeta de construcción.

```text
echo -e "package cardano-crypto-praos\n flags: -external-libsodium-vrf" > cabal.project.local
sed -i $HOME/.cabal/config -e "s/overwrite-policy:/overwrite-policy: always/g"
rm -rf $HOME/git/cardano-node/dist-newstyle/build/x86_64-linux/ghc-8.6.5
```

Construye cardano-node desde el código fuente.

```text
cabal build cardano-cli cardano-node
```

El proceso de construcción puede tomar unos minutos e incluso algunas horas dependiendo del poder del procesador de tu compoutadora.

Copia los archivos **cardano-cli** y **cardano-node** a tu carpeta *bin*.

```text
sudo cp $(find ~/git/cardano-node/dist-newstyle/build -type f -name "cardano-cli") /usr/local/bin/cardano-cli
sudo cp $(find ~/git/cardano-node/dist-newstyle/build -type f -name "cardano-node") /usr/local/bin/cardano-node
```

Verifica las versiones de **cardano-cli** and **cardano-node**.

```text
cardano-node version
cardano-cli version
```

## 📐 3. Configura tu nodo

Aquí conseguirás los archivos config.json, genesis.json y topology.json necesarios para configurar tu nodo.

```text
mkdir $NODE_HOME
cd $NODE_HOME
wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-byron-genesis.json
wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-topology.json
wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-shelley-genesis.json
wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-config.json
```

Ejecuta lo siguiente para modificar **config.json** y 

* actualiza ViewMode a "LiveView"
* actualiza TraceBlockFetchDecisions a "true"

```text
sed -i ${NODE_CONFIG}-config.json \
    -e "s/SimpleView/LiveView/g" \
    -e "s/TraceBlockFetchDecisions\": false/TraceBlockFetchDecisions\": true/g"
```

Actualiza las variables **.bashrc** de tu shell.

```
echo export CARDANO_NODE_SOCKET_PATH="$NODE_HOME/db/socket" >> ~/.bashrc
source ~/.bashrc
```

### 🔮 3.1 Configura el nodo productor de bloques y el nodo de relevo

Un nodo productor de bloques será configurado con varios pares de llaves necesarios para la creación de bloques \(cold keys (llaves frías), KES hot keys (llaves calientes KES) y VRF hot keys (llaves calientes VRF)\). Debe de conectarse solamente con sus nodos de relevo.

Un nodo de relevo no tendrá ninguna de las llaves y por lo tanto será incapaz de producir bloques. Estará conectada a su nodo productor de bloques respectivo, y a otros relevos y nodos externos.


![](../../../.gitbook/assets/producer-relay-diagram.png)

Prepara la estructura de las carpetas y copia loas archivos json esenciales a cada carpeta.

```text
mkdir relaynode1
mkdir relaynode2
cp *.json relaynode1
cp *.json relaynode2
```

Configura el archivo **topology.json** de tal forma que 

* solamente nodos de relevo se conecten al internet público y a tu nodo porductor de bloques
* el nodo productor de bloques se conecte solamente a tus nodos de relevo

Actualiza relaynode1 con lo siguiente. Simplemente copia y pega.

```text
cat > $NODE_HOME/relaynode1/${NODE_CONFIG}-topology.json << EOF 
 {
    "Producers": [
      {
        "addr": "127.0.0.1",
        "port": 3000,
        "valency": 2
      },
      {
        "addr": "127.0.0.1",
        "port": 3002,
        "valency": 2
      },
      {
        "addr": "relays-new.${NODE_URL}.iohk.io",
        "port": 3001,
        "valency": 2
      }
    ]
  }
EOF
```

Actualiza relaynode2 con lo siguiente. Simplemente copia y pega.

```text
cat > $NODE_HOME/relaynode2/${NODE_CONFIG}-topology.json << EOF 
 {
    "Producers": [
      {
        "addr": "127.0.0.1",
        "port": 3000,
        "valency": 2
      },
      {
        "addr": "127.0.0.1",
        "port": 3001,
        "valency": 2
      },
      {
        "addr": "relays-new.${NODE_URL}.iohk.io",
        "port": 3001,
        "valency": 2
      }
    ]
  }
EOF
```

Actualiza el nodo productor de bloques con lo siguiente. Simplemente copia y pega.

```text
cat > $NODE_HOME/${NODE_CONFIG}-topology.json << EOF 
 {
    "Producers": [
      {
        "addr": "127.0.0.1",
        "port": 3001,
        "valency": 2
      },
      {
        "addr": "127.0.0.1",
        "port": 3002,
        "valency": 2
      }
    ]
  }
EOF
```

La *valency* (valencia) le indica al nodo cuántas conexiones debe de mantener abiertas. Solamente direcciones DNS son afectadas. Si el valor es 0, la dirección es ignorada.

\*\*\*\*✨ **Consejo para la asignación de puertos:** Vas a necesitar asignar los puertos 3001 y 3002 a tu computadora. Chequea con [https://canyouseeme.org/](https://canyouseeme.org/)

## 🤖 4. Crea scripts de arranque

El script de arranque contiene todas las variables necesarias para ejecutar un nodo-cardano como la carpeta, puerto, db, path, archivos de config y archivo de topología.

Para tu nodo **nodo productor de bloques**:

```text
cat > $NODE_HOME/startBlockProducingNode.sh << EOF 
DIRECTORY=\$NODE_HOME
PORT=3000
HOSTADDR=0.0.0.0
TOPOLOGY=\${DIRECTORY}/${NODE_CONFIG}-topology.json
DB_PATH=\${DIRECTORY}/db
SOCKET_PATH=\${DIRECTORY}/db/socket
CONFIG=\${DIRECTORY}/${NODE_CONFIG}-config.json
cardano-node run --topology \${TOPOLOGY} --database-path \${DB_PATH} --socket-path \${SOCKET_PATH} --host-addr \${HOSTADDR} --port \${PORT} --config \${CONFIG}
EOF
```

Para tu **relaynode1**:

```text
cat > $NODE_HOME/relaynode1/startRelayNode1.sh << EOF 
DIRECTORY=\$NODE_HOME/relaynode1
PORT=3001
HOSTADDR=0.0.0.0
TOPOLOGY=\${DIRECTORY}/${NODE_CONFIG}-topology.json
DB_PATH=\${DIRECTORY}/db
SOCKET_PATH=\${DIRECTORY}/db/socket
CONFIG=\${DIRECTORY}/${NODE_CONFIG}-config.json
cardano-node run --topology \${TOPOLOGY} --database-path \${DB_PATH} --socket-path \${SOCKET_PATH} --host-addr \${HOSTADDR} --port \${PORT} --config \${CONFIG}
EOF
```

Para tu **relaynode2**:

```text
cat > $NODE_HOME/relaynode2/startRelayNode2.sh << EOF 
DIRECTORY=\$NODE_HOME/relaynode2
PORT=3002
HOSTADDR=0.0.0.0
TOPOLOGY=\${DIRECTORY}/${NODE_CONFIG}-topology.json
DB_PATH=\${DIRECTORY}/db
SOCKET_PATH=\${DIRECTORY}/db/socket
CONFIG=\${DIRECTORY}/${NODE_CONFIG}-config.json
cardano-node run --topology \${TOPOLOGY} --database-path \${DB_PATH} --socket-path \${SOCKET_PATH} --host-addr \${HOSTADDR} --port \${PORT} --config \${CONFIG}
EOF
```

El script **startStakePool.sh** automáticamente iniciará tus nodos de relevo y nodo productor de bloques.

```text
cat > $NODE_HOME/startStakePool.sh << EOF
#!/bin/bash
SESSION=$(whoami)
tmux has-session -t \$SESSION 2>/dev/null
if [ \$? != 0 ]; then
   # tmux attach-session -t \$SESSION
    tmux new-session -s \$SESSION -n window -d
    tmux split-window -v
    tmux split-window -h
    tmux select-pane -t \$SESSION:window.0
    tmux split-window -h
    tmux send-keys -t \$SESSION:window.0 $NODE_HOME/startBlockProducingNode.sh Enter
    tmux send-keys -t \$SESSION:window.1 htop Enter
    tmux send-keys -t \$SESSION:window.2 $NODE_HOME/relaynode1/startRelayNode1.sh Enter
    tmux send-keys -t \$SESSION:window.3 $NODE_HOME/relaynode2/startRelayNode2.sh Enter
    echo Stakepool started. \"tmux a\" to view.
fi
EOF
```

El script **stopStakePool.sh** automáticamente detendrá tus nodos de relevo y nodo productor de bloques.

```text
cat > $NODE_HOME/stopStakePool.sh << EOF
#!/bin/bash
SESSION=$(whoami)
tmux has-session -t \$SESSION 2>/dev/null
if [ \$? != 0 ]; then
        echo Stakepool not running.
else
        echo Stopped stakepool.
        tmux kill-session -t \$SESSION
fi
EOF
```

## ✅ 5. Inicia el nodo

**Oprime** Ctrl+Alt+T. Esto lanzará la terminal en una ventana.

Agrega permisos de ejecución al script, inicia tu stake pool, y comienza a sincronizarte con la blockchain.

```text
cd $NODE_HOME
chmod +x startBlockProducingNode.sh
chmod +x relaynode1/startRelayNode1.sh
chmod +x relaynode2/startRelayNode2.sh
chmod +x startStakePool.sh
chmod +x stopStakePool.sh
./startStakePool.sh
```

Tu stake pool se está ejecutando en una sesión **tmux**. Para adjuntarla a la termina, ejecuta lo siguiente.

```text
tmux a
```

Maximiza la ventana para un mejor vista de la sesión.

![](../../../.gitbook/assets/adatmux.png)

Para separarte de una sesión **tmux**,

**Oprime** Ctrl+b+d.

✨ Consejos para usar **tmux** con los scripts \[start\|stop\]StakePool.sh

* **Ctrl + b + arrow key** para navegar entre paneles
* **Ctrl + b + z** para hacer zoom

!Felicidades! Ahora tu nodo está operando exitosamente. Deja que se sincronice por completo.

## ⚙ 6. Crea las llaves para el nodo productor de bloques

El nodo productor de bloques requiere que crees 3 llaves definidas en las [especificaciones del libro de Shelley](https://hydra.iohk.io/build/2473732/download/1/ledger-spec.pdf):

* llave fría del stake pool
* llave caliente del stake pool \(llave KES\)
* llave VRF del stake pool

Primero, crea una par de llaves KES.

```text
cd $NODE_HOME
cardano-cli shelley node key-gen-KES \
    --verification-key-file kes.vkey \
    --signing-key-file kes.skey
```


Las llaves KES \(key evolving signature (llave evolutiva de firmas\) son creadas para asegurar tu stake pool contra hackers que quieran comprometer tus llaves. Estas deberán de ser regeneradas cada 90 días.

Las **llaves frías** deberáin de ser creadas y almacenadas en una máquina sin conexión a internet de ningún tipo. Copia el binario `cardano-cli` y ejecuta el comando `node key-gen`. Las llaves frías son los archivos almacenados en `~/cold-keys.`

Crea una carpeta para alamcenar tus llaves frías.

```text
mkdir ~/cold-keys
pushd ~/cold-keys
```

Crea un set de llaves frías y crea el archivo contador frío.

```text
cardano-cli shelley node key-gen \
    --cold-verification-key-file node.vkey \
    --cold-signing-key-file node.skey \
    --operational-certificate-issue-counter node.counter
```

Asegúrate de **respaldar todas tus llaves** a otro dispositivo de almacenamiento seguro. Crea varias copias.

Determina el número de slots por periodo KES usando el archivo génesis.

```text
pushd +1
slotsPerKESPeriod=$(cat $NODE_HOME/${NODE_CONFIG}-shelley-genesis.json | jq -r '.slotsPerKESPeriod')
echo slotsPerKESPeriod: ${slotsPerKESPeriod}
```

**Antes de continuar, tu nodo debe de estar completamente sincronizado a la blockchain. De lo contrario, no podrás calcular el periodo KES actual. Tu nodo está sincronizado cuando la _epoch_ y _slot\#_ son iguales a los que se encuentran en un explorador de bloques como [https://pooltool.io/](https://pooltool.io/)**

```text
slotNo=$(cardano-cli shelley query tip $NETWORK_IDENTIFIER | jq -r '.slotNo')
echo slotNo: ${slotNo}
```

Encuentra el kesPeriod dividiendo el número del slot tip por el slotsPerKESPeriod.

```text
kesPeriod=$((${slotNo} / ${slotsPerKESPeriod}))
echo kesPeriod: ${kesPeriod}
```

Con esté cálculo, puedes crear un certificado funcional para tu pool.

Los operadores de stake pool debe de mostrar un certificado funcional para verificar que el pool tiene la autoridad para operar. El certificado incluye la firma del operador e incluye información clave sobre el pool \(direcciones, llaves, etc.\). Los certificados fucnionales representan el enlace entre las llaves frías del operador y su llave funcional.

```text
cardano-cli shelley node issue-op-cert \
    --kes-verification-key-file kes.vkey \
    --cold-signing-key-file ~/cold-keys/node.skey \
    --operational-certificate-issue-counter ~/cold-keys/node.counter \
    --kes-period $kesPeriod \
    --out-file node.cert
```

Crea un par de llaves VRF.

```text
cardano-cli shelley node key-gen-VRF \
    --verification-key-file vrf.vkey \
    --signing-key-file vrf.skey
```

Abre una terminal en una nueva ventana con Ctrl+Alt+T y detén tu stake pool ejecutando lo siguiente:

```text
cd $NODE_HOME
./stopStakePool.sh
```

Actualiza tu script de arranque con los nuevos datos **KES, VRF y Certificado Funcional**

```text
cat > $NODE_HOME/startBlockProducingNode.sh << EOF 
DIRECTORY=\$NODE_HOME
PORT=3000
HOSTADDR=0.0.0.0
TOPOLOGY=\${DIRECTORY}/${NODE_CONFIG}-topology.json
DB_PATH=\${DIRECTORY}/db
SOCKET_PATH=\${DIRECTORY}/db/socket
CONFIG=\${DIRECTORY}/${NODE_CONFIG}-config.json
KES=\${DIRECTORY}/kes.skey
VRF=\${DIRECTORY}/vrf.skey
CERT=\${DIRECTORY}/node.cert
cardano-node run --topology \${TOPOLOGY} --database-path \${DB_PATH} --socket-path \${SOCKET_PATH} --host-addr \${HOSTADDR} --port \${PORT} --config \${CONFIG} --shelley-kes-key \${KES} --shelley-vrf-key \${VRF} --shelley-operational-certificate \${CERT}
EOF
```

Para operar un stake pool, dos sets de llaves son necesarios: la llave KES \(caliente\) ya la llave fría. Las llaves frías generan nuevas llaves calientes de manera periódica.

Ahora inicia tu stake pool.

```text
cd $NODE_HOME
./startStakePool.sh
tmux a
```

## 🔐 7. Prepara las llaves de pago y de staking

Primero, obtén los parámetros del protocolo.

Espera a que el nodo productor de bloques comience a sincronizarse antes de continuar si recibes este mensaje de error.

`cardano-cli: Network.Socket.connect: : does not exist (No such file or directory)`

```text
cardano-cli shelley query protocol-parameters \
    $NETWORK_IDENTIFIER \
    --out-file params.json
```

Las llaves de pago son usadas para enviar y recibir pagos y las llaves de staking son usadas para manejar las delegaciones del stake.

Hay dos formas de crear tus pares de llaves `payment` y `stake`. Escoge la que mejor te parezca.

Crea un nuevo para de llaves de pago:  `payment.skey` & `payment.vkey`

```text
cardano-cli shelley address key-gen \
    --verification-key-file payment.vkey \
    --signing-key-file payment.skey
```

 **MÉTODO CLI**

 Crea un nuevo par de llaves de dirección de stake: `stake.skey` & `stake.vkey`

```text
cardano-cli shelley stake-address key-gen \
    --verification-key-file stake.vkey \
    --signing-key-file stake.skey
```

Crea tu dirección de stake desde la llave de verificación de dirección de stake y almacénala en `stake.addr`

```text
cardano-cli shelley stake-address build \
    --staking-verification-key-file stake.vkey \
    --out-file stake.addr \
    --mainnet
```

Crea una dirección de pago para tu llave de pago `payment.vkey` la cual delegará a tu dirección de stake, `stake.vkey`

```text
cardano-cli shelley address build \
    --payment-verification-key-file payment.vkey \
    --staking-verification-key-file stake.vkey \
    --out-file payment.addr \
    --mainnet
```

**MÉTODO MNEMMOTÉCNICO**
Créditos a [ilap](https://gist.github.com/ilap/3fd57e39520c90f084d25b0ef2b96894) por crear este proceso.

**Beneficios**: Monitorea y controla las recompensas del pool desde cualquier billetera \(Daedalus, Yoroi o cualquier otra billetera\) que soporte staking.

Crea un mnemotécnico de 15 o 24 palabras compatible con Shelley con [Daedalus](https://daedaluswallet.io/) or [Yoroi](../../../wallets/browser-wallets/yoroi-wallet-cardano.md) en una máquina desconectada del internet.

Descarga `cardano-wallet`.

```text
cd $NODE_HOME
wget https://hydra.iohk.io/build/3662127/download/1/cardano-wallet-shelley-2020.7.28-linux64.tar.gz
tar -xvf cardano-wallet-shelley-2020.7.28-linux64.tar.gz
rm cardano-wallet-shelley-2020.7.28-linux64.tar.gz
```

Crea el script `extractPoolStakingKeys.sh`.

```text
cat > extractPoolStakingKeys.sh << HERE
#!/bin/bash 

CADDR=\${CADDR:=\$( which cardano-address )}
[[ -z "\$CADDR" ]] && ( echo "cardano-address no puede ser encontrado, saliendo..." >&2 ; exit 127 )

CCLI=\${CCLI:=\$( which cardano-cli )}
[[ -z "\$CCLI" ]] && ( echo "cardano-cli no puede ser encontrado, saliendo..." >&2 ; exit 127 )

#[[ "\$#" -ne 16 && "\$#" -ne 28 ]] && {
#       	echo "usage: `basename \$0` <ouptut dir> <15-word length mnemonic>" >&2 
#       	exit 127
#}
OUT_DIR="\$1"
[[ -e "\$OUT_DIR"  ]] && {
       	echo "El \"\$OUT_DIR\" ya existe, bórralo y ejecuta de nuevo." >&2 
       	exit 127
} || mkdir -p "\$OUT_DIR" && pushd "\$OUT_DIR" >/dev/null

shift
MNEMONIC="\$*"

# Generate the master key from mnemonics and derive the stake account keys 
# as extended private and public keys (xpub, xprv)
echo "\$MNEMONIC" |\
"\$CADDR" key from-recovery-phrase Shelley > root.prv

cat root.prv |\
"\$CADDR" key child 1852H/1815H/0H/2/0 > stake.xprv

cat root.prv |\
"\$CADDR" key child 1852H/1815H/0H/0/0 > payment.xprv

TESTNET=0
MAINNET=1
NETWORK=\$MAINNET

cat payment.xprv |\
"\$CADDR" key public | tee payment.xpub |\
"\$CADDR" address payment --network-tag \$NETWORK |\
"\$CADDR" address delegation \$(cat stake.xprv | "\$CADDR" key public | tee stake.xpub) |\
tee base.addr_candidate |\
"\$CADDR" address inspect
echo "Generated from 1852H/1815H/0H/{0,2}/0"
cat base.addr_candidate
echo

# XPrv/XPub conversion to normal private and public key, keep in mind the 
# keypars are not a valind Ed25519 signing keypairs.
TESTNET_MAGIC="--testnet-magic 42"
MAINNET_MAGIC="--mainnet"
MAGIC="\$MAINNET_MAGIC"

SESKEY=\$( cat stake.xprv | bech32 | cut -b -128 )\$( cat stake.xpub | bech32)
PESKEY=\$( cat payment.xprv | bech32 | cut -b -128 )\$( cat payment.xpub | bech32)

cat << EOF > stake.skey
{
    "type": "StakeExtendedSigningKeyShelley_ed25519_bip32",
    "description": "",
    "cborHex": "5880\$SESKEY"
}
EOF

cat << EOF > payment.skey
{
    "type": "PaymentExtendedSigningKeyShelley_ed25519_bip32",
    "description": "Payment Signing Key",
    "cborHex": "5880\$PESKEY"
}
EOF

"\$CCLI" shelley key verification-key --signing-key-file stake.skey --verification-key-file stake.evkey
"\$CCLI" shelley key verification-key --signing-key-file payment.skey --verification-key-file payment.evkey

"\$CCLI" shelley key non-extended-key --extended-verification-key-file payment.evkey --verification-key-file payment.vkey
"\$CCLI" shelley key non-extended-key --extended-verification-key-file stake.evkey --verification-key-file stake.vkey


"\$CCLI" shelley stake-address build --stake-verification-key-file stake.vkey \$MAGIC > stake.addr
"\$CCLI" shelley address build --payment-verification-key-file payment.vkey \$MAGIC > payment.addr
"\$CCLI" shelley address build \
    --payment-verification-key-file payment.vkey \
    --stake-verification-key-file stake.vkey \
    \$MAGIC > base.addr

echo "Importante, la base.addr y la base.addr_candidate deben de ser iguales"
diff base.addr base.addr_candidate
popd >/dev/null
HERE
```

Agrega los permisos y actualiza el PATH.

```text
chmod +x extractPoolStakingKeys.sh
echo PATH="$(pwd)/cardano-wallet-shelley-2020.7.28:$PATH" >> ~/.bashrc
source ~/.bashrc
```

Extrae tus kkaves. Actualiza el comando con tu frase mnemotécnica.

```text
./extractPoolStakingKeys.sh extractedPoolKeys/ <15-word length mnemonic>
```

**Importante, la base.addr y la base.addr\_candidate deben de ser iguales. Revisá el output de la pantalla. Base.addr es la dirección que será acreditada.**

Tus nuevas llaves de staking están en la carpeta `extractedPoolKeys/`

Ahora mueve los pares de llaves `payment/stake` hacia tu `$NODE_HOME` para usarlas con tu stake pool.

```text
cd extractedPoolKeys/
cp stake.vkey stake.skey stake.addr payment.vkey payment.skey base.addr $NODE_HOME
cd $NODE_HOME
#Rename to base.addr file to payment.addr
mv base.addr payment.addr
```

Genial, ahora puedes monitorear tus recompensas del pool en tu billetera.

El siguiente paso es acreditar tu dirección de pago. 

La dirección de pago puede ser acreditada desde

* tu billetera de Daedalus / Yoroi 
* si formaste parte de la ITN, puedes convertir tus llaves.

Ejecuta lo siguiente para encontrar tu dirección de pago.

```text
cat payment.addr
```

Después de haber acreditado a tu cuenta, chequea el saldo de tu dirección de pago.

Antes de continnuar, tus nodos deben de estar completamente sincronizados con la blockchain. de lo contrario, no podrás ver tus fondos.

```text
cardano-cli shelley query utxo \
    --address $(cat payment.addr) \
    $NETWORK_IDENTIFIER
```

Deberías de ver un output similar a este. Esta es la transacción de tu saldo no utilizado \(UXTO\).

```text
                           TxHash                                 TxIx        Lovelace
----------------------------------------------------------------------------------------
100322a39d02c2ead....                                              0        1000000000
```

## 👩💻 8. Registra tu dirección de stake

Crea tu certificado, `stake.cert`, usando la llave `stake.vkey`

```text
cardano-cli shelley stake-address registration-certificate \
    --staking-verification-key-file stake.vkey \
    --out-file stake.cert
```

Vas a necesitar encontrar el **tip** de la blockchain para establecer el parámetro **ttl**.

```
currentSlot=$(cardano-cli shelley query tip $NETWORK_IDENTIFIER | jq -r '.slotNo')
echo Current Slot: $currentSlot
```

Encuentra tu saldo y **UTXOs**.

```text
cardano-cli shelley query utxo \
    --address $(cat payment.addr) \
    $NETWORK_IDENTIFIER > fullUtxo.out

tail -n +3 fullUtxo.out | sort -k3 -nr > balance.out

cat balance.out

tx_in=""
total_balance=0
while read -r utxo; do
    in_addr=$(awk '{ print $1 }' <<< "${utxo}")
    idx=$(awk '{ print $2 }' <<< "${utxo}")
    utxo_balance=$(awk '{ print $3 }' <<< "${utxo}")
    total_balance=$((${total_balance}+${utxo_balance}))
    echo TxHash: ${in_addr}#${idx}
    echo ADA: ${utxo_balance}
    tx_in="${tx_in} --tx-in ${in_addr}#${idx}"
done < balance.out
txcnt=$(cat balance.out | wc -l)
echo Total ADA balance: ${total_balance}
echo Number of UTXOs: ${txcnt}
```

Encuentra el valor del keyDeposit.

```text
keyDeposit=$(cat $NODE_HOME/params.json | jq -r '.keyDeposit')
echo keyDeposit: $keyDeposit
```

El registro del certificado de la dirección de stake \(keyDeposit\) cuesta 2000000 lovelace.

Ejectura el comando build-raw

El valor del **ttl** debe de ser mayor que el tip actual. En este ejemplo, usamos el slot actual + 10000.

```text
cardano-cli shelley transaction build-raw \
    ${tx_in} \
    --tx-out $(cat payment.addr)+0 \
    --ttl $(( ${currentSlot} + 10000)) \
    --fee 0 \
    --out-file tx.tmp \
    --certificate stake.cert
```

Calcula el costo mínimo actual:

```text
fee=$(cardano-cli shelley transaction calculate-min-fee \
    --tx-body-file tx.tmp \
    --tx-in-count ${txcnt} \
    --tx-out-count 1 \
    $NETWORK_IDENTIFIER \
    --witness-count 2 \
    --byron-witness-count 0 \
    --protocol-params-file params.json | awk '{ print $1 }')
echo fee: $fee
```

Asegúrate que tu saldo sea mayor al costo a pagar + keyDeposit o esto no funcionará.

Calcula el output de tu cambio.

```text
txOut=$((${total_balance}-${keyDeposit}-${fee}))
echo Change Output: ${txOut}
```

Construye tu transacción con la cual vas a registrar tu dirección de stake.

```text
cardano-cli shelley transaction build-raw \
    ${tx_in} \
    --tx-out $(cat payment.addr)+${txOut} \
    --ttl $(( ${currentSlot} + 10000)) \
    --fee ${fee} \
    --certificate-file stake.cert \
    --out-file tx.raw
```

Firma la transacción con ambas llaves de pago y de stake.

```text
cardano-cli shelley transaction sign \
    --tx-body-file tx.raw \
    --signing-key-file payment.skey \
    --signing-key-file stake.skey \
    $NETWORK_IDENTIFIER \
    --out-file tx.signed
```

Enviá la transacción firmada.

```text
cardano-cli shelley transaction submit \
    --tx-file tx.signed \
    $NETWORK_IDENTIFIER
```

## 📄 9. Registrá tu stake pool

Creá los metadatos de tu pool con un archivo JSON. Actualizalos con la información de tu pool.

**ticker** de 3-5 caracteres.

**descripción** no puede exceder los 255 caracteres.

```text
cat > poolMetaData.json << EOF
{
"name": "NombreDeMiPool",
"description": "La descripción de mi pool",
"ticker": "NDMP",
"homepage": "https://mipoolesgenial.com"
}
EOF
```

Calcula el hash del archivo de tus metadatos.

```text
cardano-cli shelley stake-pool metadata-hash --pool-metadata-file poolMetaData.json > poolMetaDataHash.txt
```

Ahora sube el archivo **poolMetaData.json** a tu ditio web o a un si'tio público como [https://pages.github.com/](https://pages.github.com/)

Encuentra el costo operacional mínimo del pool.

```text
minPoolCost=$(cat $NODE_HOME/params.json | jq -r .minPoolCost)
echo minPoolCost: ${minPoolCost}
```

minPoolCost es de 340000000 lovelace o 340 ADA. Por lo tanto, tu `--pool-cost` debe de ser esta cantidad como mínimo.

Crea el certificado de registro para tu stake pool. Actualízalo con tu **URL de metadatos** y**dirección IP del nodo de relevo**.

**metadata-url** no debe de exceder los 64 caracteres.

```text
cardano-cli shelley stake-pool registration-certificate \
    --cold-verification-key-file ~/cold-keys/node.vkey \
    --vrf-verification-key-file vrf.vkey \
    --pool-pledge 100000000 \
    --pool-cost 345000000 \
    --pool-margin 0.15 \
    --pool-reward-account-verification-key-file stake.vkey \
    --pool-owner-stake-verification-key-file stake.vkey \
    $NETWORK_IDENTIFIER \
    --pool-relay-port 3001 \
    --pool-relay-ipv4 <your relay IP address> \
    --metadata-url <url where you uploaded poolMetaData.json> \
    --metadata-hash $(cat poolMetaDataHash.txt) \
    --out-file pool.cert
```

Aquí vamos a pledge 100 ADA con un costo fijo de ADA y un margen del pool de 15%. 

Vamos a stake nuestro pledge al pool.

```text
cardano-cli shelley stake-address delegation-certificate \
    --staking-verification-key-file stake.vkey \
    --cold-verification-key-file ~/cold-keys/node.vkey \
    --out-file deleg.cert
```

Esto crea un certificado de delegación el cual delega fondos de todas las direcciones de stake asociadas con la llave `stake.vkey` al pool perteneciente a la llave fría `node.vkey`

El compromiso de stake que hace el operador del stake pool para delegar a su propio pool se llama **Pledge**.

* Tu saldo debe de ser mayor a la cantidad de pledge.
* Tus fondos del pledge no son movidos a ningún lado. En el ejemplo de esta guía, el pledge permanece en las llaves del dueño del stake pool, específicamente `payment.addr`
* Noi complir con tu pledge resultará en que pierdas oportunidades para producir bloques y tus delegadores perderás recompensas. 
* Tu pledge no está bloqueada. Eres libre de transferir tus fondos.

Tunecesitas encontrar el **tip** de la blockchain para establecer el parámetro **ttl** de forma correcta.

```
currentSlot=$(cardano-cli shelley query tip $NETWORK_IDENTIFIER | jq -r '.slotNo')
echo Current Slot: $currentSlot
```

Encuentra tu balance y **UTXOs**.

```text
cardano-cli shelley query utxo \
    --address $(cat payment.addr) \
    $NETWORK_IDENTIFIER > fullUtxo.out

tail -n +3 fullUtxo.out | sort -k3 -nr > balance.out

cat balance.out

tx_in=""
total_balance=0
while read -r utxo; do
    in_addr=$(awk '{ print $1 }' <<< "${utxo}")
    idx=$(awk '{ print $2 }' <<< "${utxo}")
    utxo_balance=$(awk '{ print $3 }' <<< "${utxo}")
    total_balance=$((${total_balance}+${utxo_balance}))
    echo TxHash: ${in_addr}#${idx}
    echo ADA: ${utxo_balance}
    tx_in="${tx_in} --tx-in ${in_addr}#${idx}"
done < balance.out
txcnt=$(cat balance.out | wc -l)
echo Total ADA balance: ${total_balance}
echo Number of UTXOs: ${txcnt}
```

Encuentra el costo del depósito del pool.

```text
poolDeposit=$(cat $NODE_HOME/params.json | jq -r '.poolDeposit')
echo poolDeposit: $poolDeposit
```

Ejecuta el comando de transacciones build-raw.

El valor del **ttl** debe de ser mayor al tip actual. En este ejemplo, usamos el slot + 10000. 

```text
cardano-cli shelley transaction build-raw \
    ${tx_in} \
    --tx-out $(cat payment.addr)+$(( ${total_balance} - ${poolDeposit}))  \
    --ttl $(( ${currentSlot} + 10000)) \
    --fee 0 \
    --certificate-file pool.cert \
    --certificate-file deleg.cert \
    --out-file tx.tmp
```

Calcula el costo mínimo:

```text
fee=$(cardano-cli shelley transaction calculate-min-fee \
    --tx-body-file tx.tmp \
    --tx-in-count ${txcnt} \
    --tx-out-count 1 \
    $NETWORK_IDENTIFIER \
    --witness-count 3 \
    --byron-witness-count 0 \
    --protocol-params-file params.json | awk '{ print $1 }')
echo fee: $fee
```

Asegúrate que tu saldo sea mayor al costo a pagar + minPoolCost o esto no funcionará.

Calcula el output de tu cambio.

```text
txOut=$((${total_balance}-${poolDeposit}-${fee}))
echo txOut: ${txOut}
```

Construye la transacción.

```text
cardano-cli shelley transaction build-raw \
    ${tx_in} \
    --tx-out $(cat payment.addr)+${txOut} \
    --ttl $(( ${currentSlot} + 10000)) \
    --fee ${fee} \
    --certificate-file pool.cert \
    --certificate-file deleg.cert \
    --out-file tx.raw
```

Firma la transacción.

```text
cardano-cli shelley transaction sign \
    --tx-body-file tx.raw \
    --signing-key-file payment.skey \
    --signing-key-file ~/cold-keys/node.skey \
    --signing-key-file stake.skey \
    $NETWORK_IDENTIFIER \
    --out-file tx.signed
```

Envía la transacción.

```text
cardano-cli shelley transaction submit \
    --tx-file tx.signed \
    $NETWORK_IDENTIFIER
```

## 🐣 10. Localiza tu Stake pool ID y verifica que todo funcione 

Puedes obtener tu stake pool ID de la siguiente forma:

```text
cardano-cli shelley stake-pool id --verification-key-file ~/cold-keys/node.vkey > stakepoolid.txt
cat stakepoolid.txt
```

Ahora que tienes tu stake pool ID, verifica que esté incluida en la blockchain.

```text
cardano-cli shelley query ledger-state $NETWORK_IDENTIFIER | grep publicKey | grep $(cat stakepoolid.txt)
```

¡El output de una cadena no-vacía significa que estás registrado! 👏 

Con tu stake pool ID, ahora puedes econtrar tus data en los exploradores de bloques como [https://pooltool.io/](https://pooltool.io/)

## ⚙ 11. Configura tus archivos de topología

Shelley testnet ha sido lanzada sin descubrimiento de nodos peer-to-peer \(p2p\), así que esto significa que necesitarás agregar nodos de relevo confiables de maner manual para poder configurar nuestra topología. Este es un **paso crítico** ya que saltarse este paso resultará que tus bloques producidos sea aislados del resto de la red.

Hay dos formas de configurar tus archivos de topología. 

* **método topologyUpdater.sh** es automatizado y funciona después de 4 houras. 
* **método Pooltool.io** te otorga el control sobre quiénes se conectan a tus nodos.

**Método topologyUpdater.sh**

### 🚀 Publicando tu Nodo de Relevo con topologyUpdater.sh

Créditos a [GROWPOOL](https://twitter.com/PoolGrow) por la adición y créditos a [CNTOOLS Guild OPS](https://cardano-community.github.io/guild-operators/Scripts/topologyupdater.html) por la creación del proceso.

Crea el script `topologyUpdater.sh` el cual publica la información de tu nodo a una lista de búsqueda de topología.

```text
cat > $NODE_HOME/topologyUpdater.sh << EOF
#!/bin/bash
# shellcheck disable=SC2086,SC2034
 
USERNAME=$(whoami)
CNODE_PORT=3001  # debe ser igual al puerto de tu nodo de relevo establecido en el comando de arranque
CNODE_HOSTNAME="CHANGE ME"  # opcional. debe resolverse al IP del cual estás solicitando
CNODE_BIN="/usr/local/bin"
CNODE_HOME=$NODE_HOME
CNODE_LOG_DIR="\${CNODE_HOME}/logs"
GENESIS_JSON="\${CNODE_HOME}/${NODE_CONFIG}-shelley-genesis.json"
NETWORKID=\$(jq -r .networkId \$GENESIS_JSON)
CNODE_VALENCY=1   # optional for multi-IP hostnames
NWMAGIC=\$(jq -r .networkMagic < \$GENESIS_JSON)
[[ "\${NETWORKID}" = "Mainnet" ]] && HASH_IDENTIFIER="--mainnet" || HASH_IDENTIFIER="--testnet-magic \${NWMAGIC}"
[[ "\${NWMAGIC}" = "764824073" ]] && NETWORK_IDENTIFIER="--mainnet" || NETWORK_IDENTIFIER="--testnet-magic \${NWMAGIC}"
 
export PATH="\${CNODE_BIN}:\${PATH}"
export CARDANO_NODE_SOCKET_PATH="\${CNODE_HOME}/db/socket"
 
blockNo=\$(cardano-cli shelley query tip \${NETWORK_IDENTIFIER} | jq -r .blockNo )
 
# Nota:
# si ejecutas tu nodo en IPv4/IPv6 con configuración de red dual stack y solamente quieres anunciar la
# dirección IPv4 por favor agregael parámetro -4 parameter al comando curl que se muestra acontinuación (curl -4 -s ...)
if [ "\${CNODE_HOSTNAME}" != "CHANGE ME" ]; then
  T_HOSTNAME="&hostname=\${CNODE_HOSTNAME}"
else
  T_HOSTNAME=''
fi

if [ ! -d \${CNODE_LOG_DIR} ]; then
  mkdir -p \${CNODE_LOG_DIR};
fi
 
curl -s "https://api.clio.one/htopology/v1/?port=\${CNODE_PORT}&blockNo=\${blockNo}&valency=\${CNODE_VALENCY}&magic=\${NWMAGIC}\${T_HOSTNAME}" | tee -a \$CNODE_LOG_DIR/topologyUpdater_lastresult.json
EOF
```

Agrega los permisos y ejecuta el script actulizador.

```text
cd $NODE_HOME
chmod +x topologyUpdater.sh
./topologyUpdater.sh
```

Cuando el script `topologyUpdater.sh` se ejecute correctament, verás 

> `{ "resultcode": "201", "datetime":"2020-07-28 01:23:45", "clientIp": "1.2.3.4", "iptype": 4, "msg": "nice to meet you" }`

Cada vez que el script se ejecute y actualice tu IP, un registro es creado en **`$NODE_HOME/logs`**

Agrega a crontab job para automáticamente ejecutar `topologyUpdater.sh` cada hora en el minuto 22. Puedes cambiar el valor 22 con el de tu preferencia.

```text
cat > $NODE_HOME/crontab-fragment.txt << EOF
22 * * * * ${NODE_HOME}/topologyUpdater.sh
EOF
crontab -l | cat - crontab-fragment.txt >crontab.txt && crontab crontab.txt
rm crontab-fragment.txt
```

Luego de cuatro horas y cuatro actulizaciones, la IP de tu nodo será registrada en la lista de búsqueda de topología.

### 🤹♀ Actualiza los archivos topológicos de tu nodo de relevo

Completa esta sección luego de **four hours** cuando la IP de tu nodo de relevo sea registrada apropiadamente.

Crea el script `relay-topology_pull.sh` el cual busca tus colegas en tu nodo de relevo y los actualiza en tu archivo de topología.

```text
cat > $NODE_HOME/relay-topology_pull.sh << EOF
#!/bin/bash
BLOCKPRODUCING_IP=127.0.0.1
BLOCKPRODUCING_PORT=3000
RELAYNODE1_IP=127.0.0.1
RELAYNODE1_PORT=3001
RELAYNODE2_IP=127.0.0.1
RELAYNODE2_PORT=3002
curl -s -o $NODE_HOME/relaynode1/${NODE_CONFIG}-topology.json "https://api.clio.one/htopology/v1/fetch/?max=20&customPeers=\${BLOCKPRODUCING_IP}:\${BLOCKPRODUCING_PORT}:2|\${RELAYNODE1_IP}:\${RELAYNODE1_PORT}|relays-new.${NODE_URL}.iohk.io:3001:2|\${RELAYNODE2_IP}:\${RELAYNODE2_PORT}"
cp $NODE_HOME/relaynode1/${NODE_CONFIG}-topology.json $NODE_HOME/relaynode2/${NODE_CONFIG}-topology.json
EOF
```

Agrega los permisos y extrae los nuevos archivos de topología.

```text
chmod +x relay-topology_pull.sh
./relay-topology_pull.sh
```

La nueva topología entra en efecto luego de reiniciar tu stake pool.

```text
./stopStakePool.sh
./startStakePool.sh
tmux a
```

**¡No olvides reiniciar tus nodos cada vez que actualices la topología!**

**Método Pooltool.io**

1. Visita [https://htn.pooltool.io/](https://htn.pooltool.io/)
2. Crea una cuenta e inicia sesión
3. Busca tu stake pool ID
4. Click ➡ **Pool Details** &gt; **Manage** &gt; **CLAIM THIS POOL**
5. Completa el nombre de tu pool y el URL del pool si cuentas con uno.
6. Completa tus **Nodos Privados** y **TUs Nodos de Relevo** de la siguiente manera.

![](../../../.gitbook/assets/ada-relay-setup-mc4.png)

Puede encontrar tu IP pública con [https://www.whatismyip.com/](https://www.whatismyip.com/) o

```text
curl http://ifconfig.me/ip
```

Agrega solicitudes de nodos o "buddies" (amigos) a cada uno de tus nodos de relevo. Asegúrate de incluir el nodo de IOHK y tus nodos privados.

La dirección del nodo de IOHK es:

```text
relays-new.cardano-mainnet.iohk.io
```

El puerto del nodo de IOHK es:

```text
3001
```

Por ejemplo, en los buddies de relaynode1 deberías de agregar **solicitudes** para

* tu BlockProducingNode privado
* tu RelayNode2 privado
* nodo de IOHK
* y cualquier otro nodo de algún buddy o amigo que encuentres o conozcas

Por ejemplor, en los buddies de relaynode2 deberías de agregar **solicitudes** para

* tu BlockProducingNode privado
* tu RelayNode1 privado
* nodo de IOHK
* y cualquier otro nodo de algún buddy o amigo que encuentres o conozcas

la conexión de un nodo de relevo no es establecida hasta que haya una solicitud y ésta sea aprovada.

Para relaynode1, crea un script get\_buddies.sh para actualizar tu archivo topology.json

```text
cat > $NODE_HOME/relaynode1/get_buddies.sh << EOF 
#!/usr/bin/env bash

# PUEDES PASAR ESTAS CADENAS COMO VARIABLES DE AMBIENTE, O EDITARLAS EN ESTE SCRIPT
if [ -z "\$PT_MY_POOL_ID" ]; then
## CAMBIA ESTAS A LA CONVENIENCIA DE TU POOL CON TU POOL ID COMO APARECE EN EL EXPLORADOR
PT_MY_POOL_ID="XXXXXXXX"
fi

if [ -z "\$PT_MY_API_KEY" ]; then
## OBTÉN ESTO DEL PERFIL DE TU CUENTA EN LA PÁGINA DE POOLTOOL
PT_MY_API_KEY="XXXXXXXX"
fi

if [ -z "\$PT_MY_NODE_ID" ]; then
## OBTÉN ESTO DE LA PESTAÑA MANAGE DE LA PÁGINA DE POOLTOOL
PT_MY_NODE_ID="XXXXXXXX"
fi

if [ -z "\$PT_TOPOLOGY_FILE" ]; then
## ESTABLECE EL PATH QUE UTILIZA TU ARCHIVO TOPOLOGY.JSON
PT_TOPOLOGY_FILE="$NODE_HOME/relaynode1/${NODE_CONFIG}-topology.json"
fi

JSON="\$(jq -n --compact-output --arg MY_API_KEY "\$PT_MY_API_KEY" --arg MY_POOL_ID "\$PT_MY_POOL_ID" --arg MY_NODE_ID "\$PT_MY_NODE_ID" '{apiKey: \$MY_API_KEY, nodeId: \$MY_NODE_ID, poolId: \$MY_POOL_ID}')"
echo "Packet Sent: \$JSON"
RESPONSE="\$(curl -s -H "Accept: application/json" -H "Content-Type:application/json" -X POST --data "\$JSON" "https://api.pooltool.io/v0/getbuddies")"
SUCCESS="\$(echo \$RESPONSE | jq '.success')"
if [ \$SUCCESS ]; then
  echo "Éxito"
  echo \$RESPONSE | jq '. | {Producers: .message}' > \$PT_TOPOLOGY_FILE
  echo "Topology almacenada en \$PT_TOPOLOGY_FILE.  Nota topology solamente tomará efecto la próxima vez que reinicies tu nodo"
else
  echo "Fracaso "
  echo \$RESPONSE | jq '.message'
fi
EOF
```

Para relaynode2, crea un script get\_buddies.sh para actualizar tu archivo topology.json

```text
cat > $NODE_HOME/relaynode2/get_buddies.sh << EOF 
#!/usr/bin/env bash

# PUEDES PASAR ESTAS CADENAS COMO VARIABLES DE AMBIENTE, O EDITARLAS EN ESTE SCRIPT
if [ -z "\$PT_MY_POOL_ID" ]; then
## CAMBIA ESTAS A LA CONVENIENCIA DE TU POOL CON TU POOL ID COMO APARECE EN EL EXPLORADOR
PT_MY_POOL_ID="XXXXXXXX"
fi

if [ -z "\$PT_MY_API_KEY" ]; then
## OBTÉN ESTO DEL PERFIL DE TU CUENTA EN LA PÁGINA DE POOLTOOL
PT_MY_API_KEY="XXXXXXXX"
fi

if [ -z "\$PT_MY_NODE_ID" ]; then
## OBTÉN ESTO DE LA PESTAÑA MANAGE DE LA PÁGINA DE POOLTOOL
PT_MY_NODE_ID="XXXXXXXX"
fi

if [ -z "\$PT_TOPOLOGY_FILE" ]; then
## ESTABLECE EL PATH QUE UTILIZA TU ARCHIVO TOPOLOGY.JSON
PT_TOPOLOGY_FILE="$NODE_HOME/relaynode2/${NODE_CONFIG}-topology.json"
fi

JSON="\$(jq -n --compact-output --arg MY_API_KEY "\$PT_MY_API_KEY" --arg MY_POOL_ID "\$PT_MY_POOL_ID" --arg MY_NODE_ID "\$PT_MY_NODE_ID" '{apiKey: \$MY_API_KEY, nodeId: \$MY_NODE_ID, poolId: \$MY_POOL_ID}')"
echo "Packet Sent: \$JSON"
RESPONSE="\$(curl -s -H "Accept: application/json" -H "Content-Type:application/json" -X POST --data "\$JSON" "https://api.pooltool.io/v0/getbuddies")"
SUCCESS="\$(echo \$RESPONSE | jq '.success')"
if [ \$SUCCESS ]; then
  echo "éXITO"
  echo \$RESPONSE | jq '. | {Producers: .message}' > \$PT_TOPOLOGY_FILE
  echo "Topology almacenada en \$PT_TOPOLOGY_FILE.   Nota topology solamente tomará efecto la próxima vez que reinicies tu nodo"
else
  echo "Fracaso "
  echo \$RESPONSE | jq '.message'
fi
EOF
```

Para cada uno de tus nodos de relevo, actualiza las siguientes variables de pooltool.io en tu scripts get\_buddies.sh

* PT\_MY\_POOL\_ID 
* PT\_MY\_API\_KEY 
* PT\_MY\_NODE\_ID

Actualiza tus scripts get\_buddies.sh con esta información.

Usa **nano** para editar tus scripts. 

`nano $NODE_HOME/relaynode1/get_buddies.sh`

`nano $NODE_HOME/relaynode2/get_buddies.sh`

Agrega los permisos de ejecución a estos scripts. Ejecuta los scripts para actualizar tus archivos de topología.

```text
cd $NODE_HOME
chmod +x relaynode1/get_buddies.sh
chmod +x relaynode2/get_buddies.sh
./relaynode1/get_buddies.sh
./relaynode2/get_buddies.sh
```

Detén y reinicia ru stake pool para que los cambios a la topología hagan efecto.

```text
./stopStakePool.sh
./startStakePool.sh
tmux a
```

En lo que tus SOLICITUDES sean aprovadas, deberás de ejecutar nuevamente el script get\_buddies.sh para extraer la data más reciente de la topología. Reinicia tus nodos de relevo posteriormente.

\*\*\*\*🔥 **Paso crítico:** Para tener listo un stake pool funcional para producir bloques, debes de ver el número de **TXs processed** incrementar. De lo contrario, revisa tu topología y asegúrate que tus buddies de relevo estén debidamente conectados e, idealmente, produciendo algunos bloques.

![](../../../.gitbook/assets/ada-tx-processed.png)

¡Felicidades! Tu stake pool ha sido registrada y está lista para producir bloques.

## 🎇 12. Revisando las Recompensas del Stake Pool

Al concluir la epoch y asumiendo que hayas producido bloques de manera exitosa, revisa de la siguiente manera:

```text
cardano-cli shelley query stake-address-info \
 --address $(cat stake.addr) \
 $NETWORK_IDENTIFIER
```

## 🔮 13. Configura tu Consola de Control con Prometheus y Grafana

Prometheus es una plataforma de monitoreo que recolecta métricas de objetivos monitoreados mediante la extracción de métricas de extremos HTTPS en estos objetivos. [Documentación oficial disponible aquí.](https://prometheus.io/docs/introduction/overview/) Grafana es una consola de control usada para visualizar la data recolectada.

###  🐣 13.1 Instalación

Instala prometheus y prometheus node exporter.

```text
sudo apt-get install -y prometheus prometheus-alertmanager prometheus-node-exporter 
```

Instala grafana.

```text
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
```

```text
echo "deb https://packages.grafana.com/oss/deb stable main" > grafana.list
sudo mv grafana.list /etc/apt/sources.list.d/grafana.list
```

```text
sudo apt-get update && sudo apt-get install -y grafana
```

Cambia el puerto de Grafana predeterminado de 3000 a 30000

** Port 3000 is used by the block-producing node.**

```text
cd /etc/grafana
sudo sed -i grafana.ini -e "s/;http_port = 3000/http_port = 30000/g"
```

Habilita los servicios para que inicien de manera automática.

```text
sudo systemctl enable grafana-server.service
sudo systemctl enable prometheus.service
sudo systemctl enable prometheus-node-exporter.service
```

Actualiza prometheus.yml ubicado en `/etc/prometheus/prometheus.yml`

```text
cat > prometheus.yml << EOF
global:
  scrape_interval:     15s # Predeterminadamente, extrae objetivos cada 15 segundos.

  # Adjunta estas etiquetas a cualquier serie de tiempo o alertas al comunicarte con
  # sistemas externos (federation, remote storage, Alertmanager).
  external_labels:
    monitor: 'codelab-monitor'

# Una configuración de extracción conteniendo exactamente un extremo a ser extraído:
# EN este caso, es el mismo Prometheus.
scrape_configs:
  # El nombre del trabajo es agregado a la etiqueta job=<job_name> a cualquier series de tiempo extraídas de esta config.
  - job_name: 'prometheus'

    static_configs:
      - targets: ['localhost:9100']
      - targets: ['localhost:12700']
        labels:
          alias: 'block-producing-node'
          type:  'cardano-node'
      - targets: ['localhost:12701']
        labels:
          alias: 'relaynode1'
          type:  'cardano-node'
      - targets: ['localhost:12702']
        labels:
          alias: 'relaynode2'
          type:  'cardano-node'
EOF
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
```

Finalmente, reinicia los servicios.

```text
sudo systemctl restart grafana-server.service
sudo systemctl restart prometheus.service
sudo systemctl restart prometheus-node-exporter.service
```

Verifica que los servicios estén ejecutándose de manera correcta:

```text
sudo systemctl status grafana-server.service prometheus.service prometheus-node-exporter.service
```

Actualiza los archivos de configuración `${NODE_CONFIG}-config.json` con los nuevos puertos de `hasEKG`  `hasPrometheus`.

```text
cd $NODE_HOME
sed -i ${NODE_CONFIG}-config.json -e "s/    12798/    12700/g" -e "s/hasEKG\": 12788/hasEKG\": 12600/g" 
sed -i relaynode1/${NODE_CONFIG}-config.json -e "s/    12798/    12701/g" -e "s/hasEKG\": 12788/hasEKG\": 12601/g" 
sed -i relaynode2/${NODE_CONFIG}-config.json -e "s/    12798/    12702/g" -e "s/hasEKG\": 12788/hasEKG\": 12602/g" 
```

Detén i reinicia tu stake pool.

```text
cd $NODE_HOME
./stopStakePool.sh
./startStakePool.sh
tmux a
```

### 📶 13.2 Configurando la Consola de Control de Grafana 

1. Abre [http://localhost:30000](http://localhost:30000) en tu explorador
2. Inicia sesión con **admin** / **admin**
3. Cambia la contraseña
4. Click en el ícono de **engrane para configuración**, luego en **Add data Source**
5. Selecciona **Prometheus**
6. Establece el **Name** a **"prometheus**" . ✨ Las minúsculas son importantes.
7. Establece el **URL** a **http://localhost:9090**
8. Click **Save & Test**
9. Click en el ícono **Create +** &gt; **Import**
10. Agrega la consola de control importando el id: **11074**
11. Click en el botón **Load**.
12. Establece **Prometheus** data source como "**prometheus**"
13. Click en el botón **Import**.

Grafana [dashboard ID 11074](https://grafana.com/grafana/dashboards/11074) es un excelente visualizador de la salud general del sistema.

![Grafana system health dashboard](../../../.gitbook/assets/grafana.png)

Importa un **Nodo de Cardano** a la consola de control

1. Click en el ícono **Create +** &gt; **Import**
2. Agrega la consola de control usando **importing via panel json. Copy the json from below.**
3. Click el botón **Import** .

```text
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": "-- Grafana --",
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "gnetId": null,
  "graphTooltip": 0,
  "id": 1,
  "links": [],
  "panels": [
    {
      "datasource": "prometheus",
      "description": "",
      "fieldConfig": {
        "defaults": {
          "custom": {},
          "decimals": 2,
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "purple",
                "value": null
              }
            ]
          },
          "unit": "d"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 5,
        "w": 6,
        "x": 0,
        "y": 0
      },
      "id": 18,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "mean"
          ],
          "fields": "",
          "values": false
        }
      },
      "pluginVersion": "7.0.3",
      "targets": [
        {
          "expr": "(cardano_node_Forge_metrics_remainingKESPeriods_int * 6 / 24 / 6)",
          "instant": true,
          "interval": "",
          "legendFormat": "Days till renew",
          "refId": "A"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "Key evolution renew left",
      "type": "stat"
    },
    {
      "datasource": "prometheus",
      "fieldConfig": {
        "defaults": {
          "custom": {},
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "red",
                "value": null
              },
              {
                "color": "#EAB839",
                "value": 12
              },
              {
                "color": "green",
                "value": 24
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 5,
        "w": 5,
        "x": 6,
        "y": 0
      },
      "id": 12,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "mean"
          ],
          "fields": "",
          "values": false
        }
      },
      "pluginVersion": "7.0.3",
      "targets": [
        {
          "expr": "cardano_node_Forge_metrics_remainingKESPeriods_int",
          "instant": true,
          "interval": "",
          "legendFormat": "KES Remaining",
          "refId": "A"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "KES remaining",
      "type": "stat"
    },
    {
      "datasource": "prometheus",
      "description": "",
      "fieldConfig": {
        "defaults": {
          "custom": {
            "align": null
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "yellow",
                "value": 460
              },
              {
                "color": "red",
                "value": 500
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 5,
        "w": 6,
        "x": 11,
        "y": 0
      },
      "id": 2,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "mean"
          ],
          "fields": "",
          "values": false
        }
      },
      "pluginVersion": "7.0.3",
      "targets": [
        {
          "expr": "cardano_node_Forge_metrics_operationalCertificateExpiryKESPeriod_int",
          "format": "time_series",
          "instant": true,
          "interval": "",
          "legendFormat": "KES Expiry",
          "refId": "A"
        },
        {
          "expr": "cardano_node_Forge_metrics_currentKESPeriod_int",
          "instant": true,
          "interval": "",
          "legendFormat": "KES current",
          "refId": "B"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "KES Perioden",
      "type": "stat"
    },
    {
      "datasource": "prometheus",
      "description": "",
      "fieldConfig": {
        "defaults": {
          "custom": {
            "align": null
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 5,
        "w": 6,
        "x": 0,
        "y": 5
      },
      "id": 10,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "mean"
          ],
          "fields": "",
          "values": false
        }
      },
      "pluginVersion": "7.0.3",
      "targets": [
        {
          "expr": "cardano_node_ChainDB_metrics_slotNum_int",
          "instant": true,
          "interval": "",
          "legendFormat": "SlotNo",
          "refId": "A"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "Slot",
      "type": "stat"
    },
    {
      "datasource": "prometheus",
      "description": "",
      "fieldConfig": {
        "defaults": {
          "custom": {
            "align": null
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 5,
        "w": 5,
        "x": 6,
        "y": 5
      },
      "id": 8,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "mean"
          ],
          "fields": "",
          "values": false
        }
      },
      "pluginVersion": "7.0.3",
      "targets": [
        {
          "expr": "cardano_node_ChainDB_metrics_epoch_int",
          "format": "time_series",
          "instant": true,
          "interval": "",
          "legendFormat": "Epoch",
          "refId": "A"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "Epoch",
      "type": "stat"
    },
    {
      "datasource": "prometheus",
      "description": "",
      "fieldConfig": {
        "defaults": {
          "custom": {},
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 5,
        "w": 6,
        "x": 11,
        "y": 5
      },
      "id": 16,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "mean"
          ],
          "fields": "",
          "values": false
        }
      },
      "pluginVersion": "7.0.3",
      "targets": [
        {
          "expr": "cardano_node_ChainDB_metrics_blockNum_int",
          "instant": true,
          "interval": "",
          "legendFormat": "Block Height",
          "refId": "A"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "Block Height",
      "type": "stat"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "prometheus",
      "description": "",
      "fieldConfig": {
        "defaults": {
          "custom": {
            "align": null
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          }
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 9,
        "w": 9,
        "x": 0,
        "y": 10
      },
      "hiddenSeries": false,
      "id": 6,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pluginVersion": "7.0.3",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "cardano_node_ChainDB_metrics_slotInEpoch_int",
          "interval": "",
          "legendFormat": "Slot in Epoch",
          "refId": "B"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Slot in Epoch",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": true,
      "dashLength": 10,
      "dashes": false,
      "datasource": "prometheus",
      "description": "",
      "fieldConfig": {
        "defaults": {
          "custom": {
            "align": null
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          }
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 4,
      "gridPos": {
        "h": 9,
        "w": 8,
        "x": 9,
        "y": 10
      },
      "hiddenSeries": false,
      "id": 20,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pluginVersion": "7.0.3",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "cardano_node_Forge_metrics_nodeIsLeader_int",
          "interval": "",
          "legendFormat": "Node is leader",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Node is Block Leader",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "decimals": null,
          "format": "none",
          "label": "Slot",
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "prometheus",
      "description": "",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 9,
        "x": 0,
        "y": 19
      },
      "hiddenSeries": false,
      "id": 14,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "cardano_node_metrics_mempoolBytes_int / 1024",
          "interval": "",
          "intervalFactor": 1,
          "legendFormat": "Memory KB",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Memory Pool",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "KBs",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "datasource": "prometheus",
      "description": "",
      "fieldConfig": {
        "defaults": {
          "custom": {},
          "decimals": 2,
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "dthms"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 9,
        "y": 19
      },
      "id": 4,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "last"
          ],
          "fields": "",
          "values": false
        }
      },
      "pluginVersion": "7.0.3",
      "targets": [
        {
          "expr": "cardano_node_metrics_upTime_ns / (1000000000)",
          "format": "time_series",
          "instant": true,
          "interval": "",
          "intervalFactor": 1,
          "legendFormat": "Server Uptime",
          "refId": "A"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "Block-Producer Uptime",
      "type": "stat"
    }
  ],
  "refresh": "5s",
  "schemaVersion": 25,
  "style": "dark",
  "tags": [],
  "templating": {
    "list": []
  },
  "time": {
    "from": "now-24h",
    "to": "now"
  },
  "timepicker": {
    "refresh_intervals": [
      "10s",
      "30s",
      "1m",
      "5m",
      "15m",
      "30m",
      "1h",
      "2h",
      "1d"
    ]
  },
  "timezone": "",
  "title": "Cardano Node",
  "uid": "bTDYKJZMk",
  "version": 1
}
```

![Cardano-node dashboard](../../../.gitbook/assets/cardano-node-grafana.png)

¡Felicidades! Prometheus y Grafana están funcionando.

## 👏 14. Agradeciemientos, Telegram de Coincashew y Material de Referencia

### 😁 14.1 Agradecimientos

"Gracias a todos +7000 y cada uno de ustedes, los hodlers de Cardano, constructores, stakers y operadores de pools por hacer del mejor futuro una realid."

👏 "Agradecimiento especial a [Kaze-Stake](https://github.com/Kaze-Stake) por las *pull requests* y contribuciones de scripts automáticos."

### \*\*\*\*💬 **14.2 Telegram Chat Channel**

Este es el Telegram de **COINCASHEW**. Es un canal en **INGLÉS** [https://t.me/coincashew](https://t.me/coincashew)

### 😊 14.3 Donaciones y Propinas

**PRIMERAMENTE, DONACIONES A COINCASHEW, LOS CREADORES DE LA ESTA GUÍA**
"Apreciamos sinceramente todas las [donaciones](../../../contact-us/donations.md). 😁 ¡Gracias por apoyar a Cardano y a nosotros!" 

**Dirección de ADA de COINCASHEW**
```text
addr1qxhazv2dp8yvqwyxxlt7n7ufwhw582uqtcn9llqak736ptfyf8d2zwjceymcq6l5gxht0nx9zwazvtvnn22sl84tgkyq7guw7q
```

**DONACIONES A THE LEGEND OF ₳DA POR LA TRADUCCIÓN DE ESTA GUÍA**
Si llegaron hasta aquí, ¡muchas felicidades por sus esfuerzos y paciencia! No fue fácil, pero justo lo logramos. Si desean apoyarnos, pueden hacerlo delegando a nuestro stake pool **The Legend of ₳da [TLOA]** O si desean hacer alguna donación, favor hacerla a la dirección que se encuentra debajo de este comentario. Y si ya están delegando con cualquier stake pool, ¡gracias por apoyar la decentralización! 😁

**Dirección de ADA de The Legend of ₳da [TLOA]**
```text
addr1q87d37clyu4ctaz5ah8lrjwzyrsrqvpt0u83wghvdl66ea6gvzmayvajt7rstn79xtcxmuxk84k3s0qypq7af74lfemsvlzktu
```

### 📚 14.4 Material de Referencia

Para más información y documentación oficial, favor fererise a los siguientes enlaces (**en inglés**):

https://docs.cardano.org/en/latest/getting-started/stake-pool-operators/index.html

https://testnets.cardano.org/en/shelley/get-started/creating-a-stake-pool/

https://github.com/input-output-hk/cardano-tutorials

https://github.com/cardano-community/guild-operators

https://github.com/gitmachtl/scripts

#### CNTools por Guild Operators

Varios operadores de pools han preguntado cómo crear una stake pool con CNTools. La [guía oficial la puedes encontrar aquí.](https://cardano-community.github.io/guild-operators/#/Scripts/cntools)

## 🛠 15. Consejos Operacionales y de Mantenimiento

### 🤖 15.1 Actualizando el certificado funcional con un nuevo Periodo KES

Es obligatorio que recrees las llaves calientes y emitas un nuevo vertificado funcional, un proceso llamado rotando las llaves KES, cuando las laves calientes hayn caducado.

**Mainnet**: Las laves KES serán válidas por 120 rotaciones o por 90 días

**Actualizando el Periodo KES**: Cuando sea tiempo de emitir un nuevo certificado funcional, ejecuta lo siguiente:

```text
cd $NODE_HOME
slotNo=$(cardano-cli shelley query tip $NETWORK_IDENTIFIER | jq -r '.slotNo')
slotsPerKESPeriod=$(cat $NODE_HOME/${NODE_CONFIG}-shelley-genesis.json | jq -r '.slotsPerKESPeriod')
kesPeriod=$((${slotNo} / ${slotsPerKESPeriod}))
chmod u+rwx ~/cold-keys
cardano-cli shelley node issue-op-cert \
    --kes-verification-key-file kes.vkey \
    --cold-signing-key-file ~/cold-keys/node.skey \
    --operational-certificate-issue-counter ~/cold-keys/node.counter \
    --kes-period ${kesPeriod} \
    --out-file node.cert
chmod a-rwx ~/cold-keys
```

\*\*\*\*✨ **Consejo:** Con tus llaves calientes creadas, puede remover el acceso a las llaves frías para mejorar la seguridad. Esto te protege contra la eliminación o alteración accidental, y el acceso no deseado. 

Para bloquear,

```text
chmod a-rwx ~/cold-keys
```

Para desbloquear,

```text
chmod u+rwx ~/cold-keys
```

### 🔥 15.2 Reseteando la instalación

¿Quieres un inicio limpio? ¿Estás reusando un servidor existente? ¿La blockchain ha sido forked?

Elimina la carpeta git, y luego renombra tu carpetas anteriores `$NODE_HOME` y `cold-keys` \(u opcionalmente, remúevelas\). Ahora puedes comenzar con esta guía desde el inicio nuevamente.

```text
rm -rf ~/git/cardano-node/ ~/git/libsodium/
mv $NODE_HOME $(basename $NODE_HOME)_backup_$(date -I)
mv ~/cold-keys ~/cold-keys_backup_$(date -I)
```

### 🌊 15.3 Reseteando las bases de datos

¿Archivos corruptos o blockchain estancada? Elimina todas las carpetas db.

```text
cd $NODE_HOME
rm -rf db
rm -rf relaynode1/db
rm -rf relaynode2/db
```

### 📝 15.4 Modificando el pledge, los costos operacionales, el margen del pool, etc.

¿Necesitas cambiar tu pledge, los costos operacionales, el margen del pool, pool IP/puerto, o los metadatos? Simplemente reenvía tu certificado de registro de tu stake pool.

Encuentra el costo mínimo del pool.

```text
minPoolCost=$(cat $NODE_HOME/params.json | jq -r .minPoolCost)
echo minPoolCost: ${minPoolCost}
```

minPoolCost es de 340000000 lovelace o 340 ADA. Por lo tanto, tu `--pool-cost` debe de ser sta cantidad como mínimo.

Si vas a modificar tu poolMetaData.json, recuerda calcular el hash de tu archivo de metadato y reenvía el archivo poolMetaData.json actualizado. Refiérete a la [sección 9 para información.](./#9-register-your-stakepool) Si estás verificando tu stake pool ID, el hash te es proporcionado por pooltool.

```text
cardano-cli shelley stake-pool metadata-hash --pool-metadata-file poolMetaData.json > poolMetaDataHash.txt
```

Actualiza con tus configuración deseada la siguiente transacción para registrar el certificado.

**metadata-url** no debe de exceder los 64 caracteres.

```text
cardano-cli shelley stake-pool registration-certificate \
    --cold-verification-key-file ~/cold-keys/node.vkey \
    --vrf-verification-key-file vrf.vkey \
    --pool-pledge 1000000000 \
    --pool-cost 345000000 \
    --pool-margin 0.20 \
    --pool-reward-account-verification-key-file stake.vkey \
    --pool-owner-stake-verification-key-file stake.vkey \
    $NETWORK_IDENTIFIER \
    --pool-relay-port 3001 \
    --pool-relay-ipv4 <your relay IP address> \
    --metadata-url <url where you uploaded poolMetaData.json> \
    --metadata-hash $(cat poolMetaDataHash.txt) \
    --out-file pool.cert
```

Aquí nuestra pledge es de 1000 ADA con un costo fijo de 345 ADA y un margen del pool de 20%. 

Vamos a stake nuestro pledge al pool.

```text
cardano-cli shelley stake-address delegation-certificate \
    --staking-verification-key-file stake.vkey \
    --cold-verification-key-file ~/cold-keys/node.vkey \
    --out-file deleg.cert
```

Necesitas encontrar el **tip** de la blockchain para establecer el parámetro **ttl** adecuadamente.

```
currentSlot=$(cardano-cli shelley query tip $NETWORK_IDENTIFIER | jq -r '.slotNo')
echo Current Slot: $currentSlot
```

Encuentra tu saldo y **UTXOs**.

```text
cardano-cli shelley query utxo \
    --address $(cat payment.addr) \
    $NETWORK_IDENTIFIER > fullUtxo.out

tail -n +3 fullUtxo.out | sort -k3 -nr > balance.out

cat balance.out

tx_in=""
total_balance=0
while read -r utxo; do
    in_addr=$(awk '{ print $1 }' <<< "${utxo}")
    idx=$(awk '{ print $2 }' <<< "${utxo}")
    utxo_balance=$(awk '{ print $3 }' <<< "${utxo}")
    total_balance=$((${total_balance}+${utxo_balance}))
    echo TxHash: ${in_addr}#${idx}
    echo ADA: ${utxo_balance}
    tx_in="${tx_in} --tx-in ${in_addr}#${idx}"
done < balance.out
txcnt=$(cat balance.out | wc -l)
echo Total ADA balance: ${total_balance}
echo Number of UTXOs: ${txcnt}
```

Ejecuta el comando de transacción build-raw.

El valor del **ttl** debe de ser mayor al tip actual. En este ejemplo, usamos el slot actual + 10000. 

```text
cardano-cli shelley transaction build-raw \
    ${tx_in} \
    --tx-out $(cat payment.addr)+${total_balance} \
    --ttl $(( ${currentSlot} + 10000)) \
    --fee 0 \
    --certificate-file pool.cert \
    --certificate-file deleg.cert \
    --out-file tx.tmp
```

Calcula el costo mínimo a pagar:

```text
fee=$(cardano-cli shelley transaction calculate-min-fee \
    --tx-body-file tx.tmp \
    --tx-in-count ${txcnt} \
    --tx-out-count 1 \
    $NETWORK_IDENTIFIER \
    --witness-count 3 \
    --byron-witness-count 0 \
    --protocol-params-file params.json | awk '{ print $1 }')
echo fee: $fee
```

Calcula el output de tu cambio.

```text
txOut=$((${total_balance}-${fee}))
echo txOut: ${txOut}
```

Construye la transacción.

```text
cardano-cli shelley transaction build-raw \
    ${tx_in} \
    --tx-out $(cat payment.addr)+${txOut} \
    --ttl $(( ${currentSlot} + 10000)) \
    --fee ${fee} \
    --certificate-file pool.cert \
    --certificate-file deleg.cert \
    --out-file tx.raw
```

Firma la transacción.

```text
cardano-cli shelley transaction sign \
    --tx-body-file tx.raw \
    --signing-key-file payment.skey \
    --signing-key-file ~/cold-keys/node.skey \
    --signing-key-file stake.skey \
    $NETWORK_IDENTIFIER \
    --out-file tx.signed
```

Envía la transacción.

```text
cardano-cli shelley transaction submit \
    --tx-file tx.signed \
    $NETWORK_IDENTIFIER
```

Los cambios tomarán efecto a partir de la siguiente epoch. Luego de la transición de la siguiente epoch, verifica que las opciones de tu pool sean las correctas.

```text
cardano-cli shelley query ledger-state $NETWORK_IDENTIFIER --out-file ledger-state.json
jq -r '.esLState._delegationState._pstate._pParams."'"$(cat stakepoolid.txt)"'"  // empty' ledger-state.json
```

### 🧩 15.5 Transfiriendo archivos via SSH

Algunos casos en los que lo necesitarás incluyen

* Descargando respaldo de tus llaves de stake/pago
* Enviando un nuevo certificado funcional al productor de bloques desde un nodo sin conexión a internet

#### Para descargar arachivos de un nodo a tu PC local

```text
ssh <USUARIO>@<DIRECCIÓN IP> -p <PUERTO-SSH>
rsync -avzhe “ssh -p <PUERTO-SSH>” <USUARIO>@<DIRECCIÓN IP>:<PATH REMITENTE DEL NODO> <PATH DESTINATARIO EN EL PC LOCAL>
```

> Ejemplo:
>
> `ssh miusuario@6.1.2.3 -p 12345`
>
> `rsync -avzhe "ssh -p 12345" miusuario@6.1.2.3:/home/miusuario/cardano-my-node/stake.vkey ./stake.vkey`

#### Para enviar archivos desde tu PC local a tu nodo

```text
ssh <USUARIO>@<DIRECCIÓN IP> -p <PUERTO-SSH>
rsync -avzhe “ssh -p <PUERTO-SSH>” <PATH REMITENTE A TU PC LOCAL> <USUARIO>@<DIRECCIÓN IP>:<PATH DESTINATARIO DEL NODO>
```

> Ejemplo:
>
> `ssh miusuario@6.1.2.3 -p 12345`
>
> `rsync -avzhe "ssh -p 12345" ./node.cert miusuario@6.1.2.3:/home/miusuario/cardano-my-node/node.cert`

### 🏃♂ 15.6 Auto-arranque con servicios systemd

#### 🍰 Beneficios de usar systemd para tu stake pool

1. Auto-arranca tu stake pool caundo la computadora se reinicia debido a mantenimiento, apagón, etc.
2. Automáticamente reinicia procesos fallido de tu stake pool.
3. Maximiza el tiempo útil y el rendimiento de tu stake pool.

#### 🛠 Intrucciones para Configurar

Antes de comenzar, asegúrate que tu pool esté detenido.

```text
$NODE_HOME/stopStakePool.sh
```

Ejecuta lo siguiente para crear una **unidad de archivo** para definir tu configuración de `cardano-stakepool.service`

```text
cat > $NODE_HOME/cardano-stakepool.service << EOF 
# The Cardano Stakepool service (part of systemd)
# file: /etc/systemd/system/cardano-stakepool.service 

[Unit]
Description     = Cardano Stakepool Service
Wants           = network-online.target
After           = network-online.target 

[Service]
User            = $(whoami)
Type            = forking
WorkingDirectory= /home/$(whoami)/
ExecStart       = $NODE_HOME/startStakePool.sh
ExecStop        = $NODE_HOME/stopStakePool.sh
ExecReload      = $NODE_HOME/stopStakePool.sh && $NODE_HOME/startStakePool.sh
Restart         = always

[Install]
WantedBy	= multi-user.target
EOF
```

Copia la unidad de archivo a `/etc/systemd/system` y otórgale permisos.

```text
sudo cp $NODE_HOME/cardano-stakepool.service /etc/systemd/system/cardano-stakepool.service
```

```text
sudo chmod 644 /etc/systemd/system/cardano-stakepool.service
```

Ejecuta lo siguiente para habilitar el auto-arranque al momento de encender la computadora y luego iniciar tu servicio del stake pool.

```text
sudo systemctl daemon-reload
sudo systemctl enable cardano-stakepool
sudo systemctl start cardano-stakepool
```

Bien hecho. Tu stake pool ahora es dirigida por la fiabilidad y solidez de systemd. Acontinuacióin encontrarás algunos comandos para usar con systemd.

#### 📎 Vuelve a adjuntarte a la sesión tmux de tu stake pool luego del inicio del sistema.

```text
tmux a
```

#### ✅ Chequear si el servicio del stake pool está activo

```text
sudo systemctl is-active cardano-stakepool
```

#### 🔎 Ver el estado del servicio del stake pool

```text
sudo systemctl status cardano-stakepool
```

#### 🔄 Reiniciar el servicio del stake pool

```text
sudo systemctl reload-or-restart cardano-stakepool
```

#### 🛑 Deteniendo el servicio del stake pool

```text
sudo systemctl stop cardano-stakepool
```

#### 🗄 Filtrando registros

```text
journalctl --unit=cardano-stakepool --since=yesterday
journalctl --unit=cardano-stakepool --since=today
journalctl --unit=cardano-stakepool --since='2020-07-29 00:00:00' --until='2020-07-29 12:00:00'
```

### ✅ 15.7 Verifica el ticker de tu stake pool con la llave de ITN

Para defender contra falsificación y apropiación de las stake pools con buena reputación, un dueño puede verificar su ticker proporcionando titularidad de un stake pool en ITN.

La fase de la Testnet Incentivada de la era Shelley de Cardano se llevó a cabo desde finales de noviembre del 2019 hasta finales de junio 2020. Si participaste, puedes verificar tu ticker.

Asegúrate que los binarios `jcli` de la ITN estén presentes en `$NODE_HOME`. Usa `jcli` para firmar tu stake pook id con tu `itn_owner.skey`

```text
./jcli key sign --secret-key itn_owner.skey stakepoolid.txt --output stakepoolid.sig
```

Visita [pooltool.io](https://pooltool.io/) e introduce tu llaver pública de propietario y los datos de tu testigo de pool id en la sección de metadatos.

Encuentra tu testigo de pool id con el siguiente comando.

```text
cat stakepoolid.sig
```

Encuentra tu llave pública de propietario en el archivo que creaste en la ITN. Estos datos pueden ser almacenados en un archivo con terminación `.pub`

Finalmete, realiza las [instrucciones de la sección 15.4 para actualizar los datos del registro de tu pool](./#15-4-changing-the-pledge-fee-margin-etc) con el **`metadata-url`** y el **`metadata-hash`** generados por pooltool . Nota que los metadatos tienen un campo "expandible" lo cual comprueba la titularidad de tu ticker desde ITN.

### 📚 15.8 Actualizando os archivos de configuración de tu nodo

Manten tus archivos de configuración recientes descargando los archivos .json más recientes.

```text
NODE_BUILD_NUM=$(curl https://hydra.iohk.io/job/Cardano/iohk-nix/cardano-deployment/latest-finished/download/1/index.html | grep -e "build" | sed 's/.*build\/\([0-9]*\)\/download.*/\1/g')
cd $NODE_HOME
wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-byron-genesis.json
wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-shelley-genesis.json
wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-config.json
sed -i ${NODE_CONFIG}-config.json \
    -e "s/SimpleView/LiveView/g" \
    -e "s/TraceBlockFetchDecisions\": false/TraceBlockFetchDecisions\": true/g"
```

## 🌜 16. Retirando tu stake pool

Encuentra los slots por epoch.

```text
epochLength=$(cat $NODE_HOME/${NODE_CONFIG}-shelley-genesis.json | jq -r '.epochLength')
echo epochLength: ${epochLength}
```

 Encuentra el slot actual consultando el tip.

```text
slotNo=$(cardano-cli shelley query tip $NETWORK_IDENTIFIER | jq -r '.slotNo')
echo slotNo: ${slotNo}
```

Calcula la epoch actual dividiendo el número del tip en el slot por la epochLength.

```text
epoch=$(( $((${slotNo} / ${epochLength})) + 1))
echo current epoch: ${epoch}
```

Encuentra el valor de eMax.

```text
eMax=$(cat $NODE_HOME/params.json | jq -r '.eMax')
echo eMax: ${eMax}
```

\*\*\*\*🚧 **Ejemplo**: si estamos en la epoch 39 y eMax es 18,

* la epoch más pronta para retirar es 40 \( epoch actual  + 1\).
* la epocj más tardía para retirar es 57 \( eMax + epoch actual\). 

Pretendamos quye deseamos retirarnos lo más pronto posible en la epoch 40.

Crea el certificado de cancelación de registro y guárdalo como `pool.dereg`:

```text
cardano-cli shelley stake-pool deregistration-certificate \
--cold-verification-key-file ~/cold-keys/node.vkey \
--epoch $((${epoch} + 1)) \
--out-file pool.dereg
echo pool will retire at end of epoch: $((${epoch} + 1))
```

Encuentra tu saldo y **UTXOs**.

```text
cardano-cli shelley query utxo \
    --address $(cat payment.addr) \
    $NETWORK_IDENTIFIER > fullUtxo.out

tail -n +3 fullUtxo.out | sort -k3 -nr > balance.out

cat balance.out

tx_in=""
total_balance=0
while read -r utxo; do
    in_addr=$(awk '{ print $1 }' <<< "${utxo}")
    idx=$(awk '{ print $2 }' <<< "${utxo}")
    utxo_balance=$(awk '{ print $3 }' <<< "${utxo}")
    total_balance=$((${total_balance}+${utxo_balance}))
    echo TxHash: ${in_addr}#${idx}
    echo ADA: ${utxo_balance}
    tx_in="${tx_in} --tx-in ${in_addr}#${idx}"
done < balance.out
txcnt=$(cat balance.out | wc -l)
echo Total ADA balance: ${total_balance}
echo Number of UTXOs: ${txcnt}
```

Ejecuta el comando de transacción build-raw.

El calor del **ttl** debe de ser mayor al tip actual. En este ejemplo, usamos el slot actual + 10000. 

```text
cardano-cli shelley transaction build-raw \
    ${tx_in} \
    --tx-out $(cat payment.addr)+${total_balance} \
    --ttl $(( ${slotNo} + 10000)) \
    --fee 0 \
    --certificate-file pool.dereg \
    --out-file tx.tmp
```

Calcula el costo a pagar:

```text
fee=$(cardano-cli shelley transaction calculate-min-fee \
    --tx-body-file tx.tmp \
    --tx-in-count ${txcnt} \
    --tx-out-count 1 \
    $NETWORK_IDENTIFIER \
    --witness-count 2 \
    --byron-witness-count 0 \
    --protocol-params-file params.json | awk '{ print $1 }')
echo fee: $fee
```

Calcula el output de tu cambio.

```text
txOut=$((${total_balance}-${fee}))
echo txOut: ${txOut}
```

Construye la transacción.

```text
cardano-cli shelley transaction build-raw \
    ${tx_in} \
    --tx-out $(cat payment.addr)+${txOut} \
    --ttl $(( ${slotNo} + 10000)) \
    --fee ${fee} \
    --certificate-file pool.dereg \
    --out-file tx.raw
```

Firma la transacción.

```text
cardano-cli shelley transaction sign \
    --tx-body-file tx.raw \
    --signing-key-file payment.skey \
    --signing-key-file ~/cold-keys/node.skey \
    $NETWORK_IDENTIFIER \
    --out-file tx.signed
```

Envía la transacción.

```text
cardano-cli shelley transaction submit \
    --tx-file tx.signed \
    $NETWORK_IDENTIFIER
```

Tu pool será retirado al final de tu epoch especificada. En este ejemplo, el retiro ocurrirá al final de la epoch 40. 

Si cambia de parecer, puede crear y envair un nuevo certificado de registro antes de que termine la epoch 40, lo cual anulará el certificado de cancelación de registro.

Luego de la epoch de retiro, puede verificar que el pool fuer retirado exitosamente utilizando el siguiente comando el cual debería de regresar una cadena vacía.

```text
cardano-cli shelley query ledger-state $NETWORK_IDENTIFIER --out-file ledger-state.json
jq -r '.esLState._delegationState._pstate._pParams."'"$(cat stakepoolid.txt)"'"  // empty' ledger-state.json
```

