# OpenVPN para Docker

Fork mantenido de [kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn), con OpenVPN **2.7.9** compilado desde el archivo fuente oficial y verificado mediante SHA-256. La imagen se basa en Alpine Linux 3.23; Easy-RSA y las dependencias se instalan desde sus repositorios.

Repositorio: [xlokius/docker-openvpn](https://github.com/xlokius/docker-openvpn)

## Estado

- Conserva el flujo de comandos, el formato del volumen `/etc/openvpn` y las funciones existentes de PKI, clientes, OTP, IPv4/IPv6 y TCP.
- `ovpn_genconfig -C` configura `data-ciphers`; el valor predeterminado es `AES-256-GCM:AES-128-GCM:CHACHA20-POLY1305`.
- Los volúmenes existentes no se migran automáticamente. Haz una copia de seguridad y prueba la actualización con clientes antes de aplicarla al servidor.
- La versión OpenVPN se fija en 2.7.9 y se comprueba su SHA-256 durante la compilación. La imagen base y los paquetes Alpine no están fijados por digest o versión; reconstrucciones posteriores pueden incluir cambios.
- `comp-lzo` sigue disponible para compatibilidad. Mantén la compresión desactivada salvo que un cliente antiguo la necesite.

## Inicio rápido

Reemplaza `VPN.SERVERNAME.COM` por el nombre DNS público del servidor.

```sh
OVPN_DATA="ovpn-data-example"
IMAGE="xlokius/docker-openvpn:2.7.9"

docker build --pull -t "$IMAGE" .
docker run --rm "$IMAGE" openvpn --version

docker volume create --name "$OVPN_DATA"
docker run -v "$OVPN_DATA:/etc/openvpn" --rm "$IMAGE" \
  ovpn_genconfig -u udp://VPN.SERVERNAME.COM
docker run -v "$OVPN_DATA:/etc/openvpn" --rm -it "$IMAGE" ovpn_initpki
docker run -v "$OVPN_DATA:/etc/openvpn" -d -p 1194:1194/udp \
  --cap-add=NET_ADMIN "$IMAGE"
```

## Publicación en Docker Hub

El workflow `.github/workflows/docker-publish.yml` construye la imagen para
`linux/amd64` en cada push. Publica una etiqueta `sha-<commit>` y, para la rama
`master`, actualiza también `latest`. Las etiquetas de Git con formato de versión,
por ejemplo `v1.0.0`, generan la etiqueta de imagen `1.0.0`. Los pull requests
construyen la imagen sin publicarla.

Para habilitar la publicación, en GitHub ve a **Settings → Secrets and variables
→ Actions** y configura:

- Variable `DOCKERHUB_USERNAME`: nombre de usuario de Docker Hub, propietario
  del repositorio de imagen `docker-openvpn`.
- Secret `DOCKERHUB_TOKEN`: token de acceso de Docker Hub con permiso de escritura.

Activa GitHub Actions en la pestaña **Actions** del fork si aún no está habilitado.
Tras publicar, sustituye `USUARIO` por el usuario de Docker Hub para descargar:

```sh
docker pull USUARIO/docker-openvpn:latest
```

Si la imagen es privada, inicia sesión en Docker Hub en el servidor antes de
descargarla. Actualizar la imagen no reinicia el contenedor automáticamente;
recréalo cuando corresponda conservando su volumen y configuración de red.

## Clientes

Crear un certificado y exportar un perfil `.ovpn` autocontenido:

```sh
docker run -v "$OVPN_DATA:/etc/openvpn" --rm -it "$IMAGE" \
  easyrsa build-client-full CLIENTNAME nopass
docker run -v "$OVPN_DATA:/etc/openvpn" --rm "$IMAGE" \
  ovpn_getclient CLIENTNAME > CLIENTNAME.ovpn
```

El archivo exportado incluye la clave privada del cliente. Guárdalo y compártelo como una credencial secreta. Para revocar un cliente:

```sh
docker run -v "$OVPN_DATA:/etc/openvpn" --rm -it "$IMAGE" \
  ovpn_revokeclient CLIENTNAME
```

Consulta [`docs/clients.md`](docs/clients.md) y [`docs/otp.md`](docs/otp.md) para más opciones.

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
