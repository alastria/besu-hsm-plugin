# Integración de un HSM con un nodo BESU de Alastria/ISBE en la red USECASE

## 1. Introducción

Este repositorio contiene un plugin para Hyperledger Besu que permite externalizar la gestión de la clave del nodo hacia un Hardware Security Module (HSM). En un entorno de Alastria/ISBE, esta capacidad es clave para proteger la identidad del nodo y para asegurar que las operaciones sensibles de consenso y de handshake de red se realicen sin exponer la clave privada en memoria o en disco.

El plugin incluido en este proyecto es compatible con HSMs que expongan un interfaz PKCS#11 y, en particular, con el modo recomendado para producción: `native-pkcs11`. Este modo permite que Besu use la clave del nodo directamente dentro del HSM, manteniendo la clave privada protegida y delegando operaciones como firma ECDSA y derivación ECDH al dispositivo seguro.

Para un nodo de la red USECASE, el objetivo es conectar el nodo BESU a un HSM genérico de forma segura y reproducible, de manera que:

- la clave de identidad del nodo viva dentro del HSM;
- el nodo pueda firmar bloques o participar en consenso sin exponer esa clave;
- la configuración del nodo quede documentada y preparada para despliegue repetible.

> Importante: la clave protegida por el HSM para este plugin es la clave del nodo BESU, utilizada para la identidad de red y para operaciones de consenso. No es la clave de cuenta que firma transacciones del usuario.

## 2. Qué necesitas recopilar antes de conectar el HSM

Antes de arrancar el nodo, conviene reunir estos datos:

- Nombre del HSM o proveedor: por ejemplo, Thales, Utimaco, Gemalto, Safenet o un HSM genérico PKCS#11.
- Ruta de la biblioteca PKCS#11 del fabricante, por ejemplo `/usr/local/lib/libCryptoki2_64.so`.
- Slot o índice de slot donde está el token del HSM.
- PIN o contraseña del token.
- Alias o etiqueta de la clave privada que Besu debe usar.
- Curva elíptica a utilizar. Para compatibilidad general, se recomienda `secp256k1`.
- La clave pública asociada a esa clave, necesaria para incluirla en la configuración de red o en el genesis del nodo.
- Datos de red del nodo USECASE: `genesis.json`, `bootnodes`, `network-id`, `p2p-port`, etc.

## 3. Preparación paso a paso para conectar un HSM genérico a un nodo BESU en la red USECASE

### Paso 1. Elegir el modo de proveedor adecuado

Para un HSM genérico compatible con PKCS#11, el modo recomendado es:

- `native-pkcs11`

Este es el modo más adecuado para producción y el que mejor encaja con la recomendación del proyecto.

### Paso 2. Preparar el archivo de configuración PKCS#11

Crea un archivo de configuración, por ejemplo en `/etc/besu/pkcs11.cfg`, con el siguiente contenido:

```text
library = /ruta/al/driver/pkcs11.so
slot = 0
```

Si tu HSM usa un índice de slot en lugar de un slot fijo, puedes usar:

```text
library = /ruta/al/driver/pkcs11.so
slotListIndex = 0
```

Sustituye la ruta y el slot por los valores reales del entorno USECASE.

### Paso 3. Preparar el archivo con el PIN del token

Crea un archivo con el PIN o la contraseña del token, por ejemplo:

```text
/etc/besu/hsm-pin.txt
```

Ejemplo:

```text
1234
```

Asegúrate de restringir permisos de lectura al archivo:

```bash
chmod 600 /etc/besu/hsm-pin.txt
```

### Paso 4. Crear o importar la clave del nodo en el HSM

Dentro del HSM, genera o importa un par de claves elípticas para el nodo. La clave debe quedar accesible por alias o etiqueta, por ejemplo:

```text
usecase-validator-01
```

El alias debe ser conocido por Besu y debe ser el mismo que se use en la configuración del arranque del nodo.

Además, debes conservar la clave pública correspondiente, porque será necesaria para configurar la identidad del validador o del nodo dentro de la red USECASE.

### Paso 5. Definir la curva elíptica

Para la red USECASE, la opción recomendada es:

```text
secp256k1
```

Esto ofrece mejor compatibilidad con la mayoría de implementaciones de red y con las funciones de descubrimiento y consenso usadas por Besu.

### Paso 6. Preparar la configuración del nodo BESU

El nodo debe arrancarse con los flags del plugin. Un ejemplo base es:

```bash
besu \
  --security-module=hsm \
  --plugin-hsm-provider-type=native-pkcs11 \
  --plugin-hsm-config-path=/etc/besu/pkcs11.cfg \
  --plugin-hsm-password-path=/etc/besu/hsm-pin.txt \
  --plugin-hsm-key-alias=usecase-validator-01 \
  --plugin-hsm-ec-curve=secp256k1 \
  --genesis-file=/etc/besu/genesis.json \
  --rpc-http-enabled \
  --rpc-http-host=0.0.0.0 \
  --rpc-http-port=8545 \
  --p2p-port=30303
```

En una red USECASE real, se añadirán también los parámetros propios de la topología, por ejemplo:

- `--bootnodes=...`
- `--network-id=...`
- `--data-path=/var/lib/besu`
- `--node-private-key-file=...` si aplica a la arquitectura concreta

### Paso 7. Asegurar que la clave pública del nodo esté asociada al rol del nodo

En una red basada en validadores o en una topología QBFT/IBFT, la clave pública del nodo debe estar incluida correctamente en la configuración del genesis o en el conjunto de validadores de la red.

Esto es esencial porque el nodo BESU no solo necesita que la clave privada esté protegida en el HSM, sino que la red debe poder verificar que esa identidad corresponde al rol esperado dentro de USECASE.

### Paso 8. Verificar que la conexión al HSM funciona

Una vez arrancado el nodo, revisa los logs para confirmar que la inicialización del plugin ha sido correcta. Debe aparecer un proceso de carga del proveedor HSM y, si todo está bien, el nodo debería arrancar normalmente y participar en la red.

También puedes comprobar el estado del nodo con comandos RPC, por ejemplo:

```bash
curl http://localhost:8545 -X POST \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_nodeInfo","params":[],"id":1}'
```

Y, si la red usa validadores, también puedes consultar:

```bash
curl http://localhost:8545 -X POST \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"qbft_getValidatorsByBlockNumber","params":["latest"],"id":1}'
```

### Paso 9. Recomendaciones operativas para USECASE

- Mantener el PIN del token fuera del código fuente y con permisos restringidos.
- Guardar el alias y la ruta de la biblioteca en un inventario de despliegue.
- Proteger el acceso físico y administrativo al HSM.
- Usar `secp256k1` salvo que la red USECASE requiera explícitamente otra curva.
- Para despliegues de producción, revisar si el HSM requiere políticas adicionales de firma, ECDH o acceso a slots concretos.

## 4. Plantilla de valores para un despliegue genérico en USECASE

Puedes rellenar esta plantilla con los datos reales del entorno:

```text
HSM_PROVIDER = native-pkcs11
HSM_LIBRARY = /ruta/al/driver/pkcs11.so
HSM_SLOT = 0
HSM_PIN_FILE = /etc/besu/hsm-pin.txt
HSM_KEY_ALIAS = usecase-validator-01
HSM_CURVE = secp256k1
BESU_DATA_DIR = /var/lib/besu
BESU_GENESIS = /etc/besu/genesis.json
BESU_BOOTNODES = <bootnode(s) de USECASE>
```

## 5. Resumen

Para conectar un HSM genérico a un nodo BESU en la red USECASE, el flujo esencial es:

1. confirmar que el HSM es compatible con PKCS#11;
2. preparar el archivo de configuración y el PIN;
3. crear o importar la clave del nodo en el HSM;
4. arrancar Besu con el plugin `native-pkcs11` y el alias de la clave;
5. verificar que el nodo participa correctamente en la red.

Este enfoque permite proteger la identidad del nodo y usar el HSM como componente de confianza para el funcionamiento del nodo en una red Alastria/ISBE como USECASE.
