# OpenVPN para Docker

Fork mantenido de [kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn), con OpenVPN **2.7.9** compilado desde el archivo fuente oficial y verificado mediante SHA-256. La imagen se basa en Alpine Linux 3.23; Easy-RSA y las dependencias se instalan desde sus repositorios.

Repositorio: [xlokius/docker-openvpn](https://github.com/xlokius/docker-openvpn)

## Estado

- Conserva el flujo de comandos, el formato del volumen `/etc/openvpn` y las funciones existentes de PKI, clientes, OTP, IPv4/IPv6 y TCP.
- `ovpn_genconfig -C` configura `data-ciphers`; el valor predeterminado es `AES-256-GCM:AES-128-GCM:CHACHA20-POLY1305`.
- Los volúmenes existentes no se migran automáticamente. Haz una copia de seguridad y prueba la actualización con clientes antes de aplicarla al servidor.
- La versión OpenVPN se fija en 2.7.9 y se comprueba su SHA-256 durante la compilación. La imagen base y los paquetes Alpine no están fijados por digest o versión; reconstrucciones posteriores pueden incluir cambios.
- `comp-lzo` sigue disponible para compatibilidad. Mantén la compresión desactivada salvo que un cliente antiguo la necesite.

## Instalación de una VPN nueva en un VPS

Esta guía usa Docker Engine en un servidor Linux `x86_64` (AMD64). Si todavía no
lo tienes, consulta la [instalación oficial para Ubuntu](https://docs.docker.com/engine/install/ubuntu/).
Los comandos usan `sudo`; si ya administras Docker como root, puedes omitirlo.

Los pasos de generación de configuración y PKI son para una instalación nueva.
Para una VPN existente, consulta **Actualizar una VPN existente** más abajo.

### 1. Descargar la imagen

```sh
sudo docker pull xlokius/docker-openvpn:latest
sudo docker run --rm xlokius/docker-openvpn:latest openvpn --version
```

### 2. Configurar el servidor y la PKI

Reemplaza `VPN.SERVERNAME.COM` por el dominio o IP pública del VPS. Conserva el
nombre del volumen: contiene la configuración, los certificados y sus claves.

```sh
OVPN_DATA="openvpn-data"
IMAGE="xlokius/docker-openvpn:latest"
VPN_HOST="VPN.SERVERNAME.COM"

sudo docker volume create "$OVPN_DATA"
sudo docker run --rm -v "$OVPN_DATA:/etc/openvpn" "$IMAGE" \
  ovpn_genconfig -u "udp://$VPN_HOST"
sudo docker run --rm -it -v "$OVPN_DATA:/etc/openvpn" "$IMAGE" ovpn_initpki
```

La inicialización solicita una contraseña para proteger la autoridad certificadora
(CA). Guárdala de forma segura; se necesita para firmar certificados de clientes.
Si abres otra terminal, vuelve a definir `OVPN_DATA` e `IMAGE` antes de usar los
comandos siguientes.

### 3. Iniciar y revisar la VPN

```sh
sudo docker run -d \
  --name openvpn \
  --restart unless-stopped \
  -p 1194:1194/udp \
  --cap-add=NET_ADMIN \
  -v "$OVPN_DATA:/etc/openvpn" \
  "$IMAGE"

sudo docker ps
sudo docker logs --tail 50 openvpn
```

Permite el puerto **1194/UDP** en el firewall del servidor y en las reglas de red
del proveedor del VPS. Si el cliente se conecta pero no llega a otras redes,
revisa el forwarding y el firewall del host: consulta [`docs/debug.md`](docs/debug.md).

### 4. Crear un perfil de cliente

Este ejemplo crea un cliente `laptop` sin contraseña para su clave privada. Para
protegerla con contraseña, omite `nopass` y sigue las indicaciones interactivas.

```sh
CLIENTE="laptop"
sudo docker run --rm -it -v "$OVPN_DATA:/etc/openvpn" "$IMAGE" \
  easyrsa build-client-full "$CLIENTE" nopass
sudo docker run --rm -v "$OVPN_DATA:/etc/openvpn" "$IMAGE" \
  ovpn_getclient "$CLIENTE" > "$CLIENTE.ovpn"
```

El archivo `laptop.ovpn` queda en el directorio actual del VPS. Contiene una
clave privada: transfiérelo por un medio seguro e impórtalo en el cliente OpenVPN.
Consulta [`docs/clients.md`](docs/clients.md) y [`docs/otp.md`](docs/otp.md) para
más opciones.

### Actualizar una VPN existente

Antes de actualizar, respalda el volumen o carpeta montada en `/etc/openvpn`,
incluidas la CA y las claves. Descarga la nueva imagen:

```sh
sudo docker pull xlokius/docker-openvpn:latest
```

Descargarla no actualiza el contenedor que ya está en ejecución. Recrea el
contenedor conservando el mismo volumen, puertos, capacidades y opciones de red;
habrá una interrupción mientras se reinicia. No ejecutes `ovpn_genconfig` ni
`ovpn_initpki` sobre los datos existentes. Revisa la configuración y prueba los
clientes después de actualizar. Si usas Compose, actualiza la imagen en tu
archivo y recrea el servicio conservando sus volúmenes.

### Construcción local opcional

Para desarrollar o probar cambios en el repositorio puedes construir la imagen
localmente desde la raíz del proyecto:

```sh
docker build --pull -t xlokius/docker-openvpn:local .
```

## Revocar un cliente

Para revocar un cliente existente, define `OVPN_DATA` e `IMAGE` como en la guía
anterior y reemplaza `CLIENTNAME` por su nombre:

```sh
sudo docker run -v "$OVPN_DATA:/etc/openvpn" --rm -it "$IMAGE" \
  ovpn_revokeclient CLIENTNAME
```

## Seguridad y actualización

- El volumen contiene la CA y las claves PKI. Restringe el acceso y mantén copias de seguridad protegidas. Para separar la CA del servidor, sigue [`docs/paranoid.md`](docs/paranoid.md).
- La inicialización de PKI solicita una contraseña para proteger la clave de la CA. Evita `nopass` para la CA.
- El contenedor requiere `NET_ADMIN` para configurar la red. No uses `--privileged` salvo que hayas confirmado que tu entorno lo exige.
- Antes de actualizar, respalda el volumen, revisa la configuración generada y confirma que los clientes sean compatibles con OpenVPN 2.7.
- La imagen no configura el firewall ni el forwarding del host. Consulta [`docs/debug.md`](docs/debug.md) si el cliente conecta pero no alcanza las redes esperadas.

## Pruebas

Las pruebas del repositorio están documentadas en [`test/README.md`](test/README.md). Ejecuta la suite antes de publicar una imagen o actualizar un servidor.

## Licencia

Consulta [`LICENSE`](LICENSE).
