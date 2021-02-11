# Dockerizando la raspberry

- [Dockerizando la raspberry](#dockerizando-la-raspberry)
  - [Requisitos](#requisitos)
    - [Creacion de acme.json y traefik.log](#creacion-de-acmejson-y-traefiklog)
    - [Creación de secretos](#creación-de-secretos)
      - [Instalación de Secrethub](#instalación-de-secrethub)
      - [Creación httpbasic auth para servicios](#creación-httpbasic-auth-para-servicios)
      - [Creación del resto de secretos](#creación-del-resto-de-secretos)
    - [Creación certificados SSL](#creación-certificados-ssl)
    - [Sacar /var/lib/docker de microSD](#sacar-varlibdocker-de-microsd)
  - [Despliegue](#despliegue)
  - [Destrucción del entorno](#destrucción-del-entorno)

## Requisitos

### Creacion de acme.json y traefik.log

```bash
touch acme.json traefik.log
chmod 600 acme.json
```

### Creación de secretos

Necesitaremos los siguientes secretos:

| **NOMBRE** | **DESCRIPCION** |
|--|--|
| `my_email` | email para la creación de certificados con Lets Encrypt |
| `transmission_user` | usuario de acceso para `TRANSMISSION_DOMAIN` |
| `traefik_pilot_token` | token de instancia de **Traefik Pilot** |
| `netdata_pass` | contraseña de acceso para `NETDATA_DOMAIN` |
| `nginx_pass` | contraseña de acceso para `NGINX_DOMAIN` |
| `traefik_pass` | contraseña de acceso a `TRAEFIK_DOMAIN` |
| `transmission_pass` | contraseña de acceso para `TRANSMISSION_DOMAIN` |
| `netdata_creds` | credenciales htpasswd para acceso a `NETDATA_DOMAIN` (__USER__: admin, __PASS__: `netdata_pass`) |
| `nginx_creds` | credenciales htpasswd para acceso a `NGINX_DOMAIN` (__USER__: admin, __PASS__: `nginx_pass`) |
| `traefik_creds` | credenciales htpasswd para acceso a `TRAEFIK_DOMAIN` (__USER__: admin, __PASS__: `traefik_pass`) |
| `netdata_domain` | dominio para **NETDATA** |
| `nginx_domain` | dominio para **NGINX** |
| `traefik_domain` | dominio para **TRAEFIK** |
| `transmission_domain` | dominio para **TRANSMISSION** |

Usaremos [Secrethub](https://secrethub.io) como herramienta de gestión de secretos

#### Instalación de Secrethub

No es posible inyectar secretos en docker como variables de entorno (se montan como volumen en `/run/secrets`), así que hay que recurrir a software de terceros. Para ello, instalaremos [Secrethub](https://secrethub.io/). Desde [este enlace](https://github.com/secrethub/secrethub-cli/releases/latest/) podemos descargar el binario para arquitectura **ARM7** (raspberry pi 3)

```bash
sudo dpkg -i secrethub-v0.41.2-linux-armv7.deb
```

A continuación, inicializaos y creamos un repo para nuestro entorno:

```bash
export SECRETHUB_USERNAME=osmollo
secrethub init --setup-code $SETUP_CODE
secrethub repo init ${SECRETHUB_USERNAME}/docker-compose
```

[Documentación Secrethub](https://secrethub.io/blog/inject-secrets-into-docker-containers/)

#### Creación httpbasic auth para servicios

En primer lugar, instalamos el paquete `apache2-utils`:

```bash
sudo apt install apache2-utils
```

Y a continuación creamos los secretos para las credenciales basicauth

```bash
secrethub generate rand ${SECRETHUB_USERNAME}/docker-compose/traefik_pass
secrethub generate rand ${SECRETHUB_USERNAME}/docker-compose/nginx_pass
secrethub generate rand ${SECRETHUB_USERNAME}/docker-compose/netdata_pass

htpasswd -nb admin  "$(secrethub read ${SECRETHUB_USERNAME}/docker-compose/traefik_pass)" | secrethub write ${SECRETHUB_USERNAME}/docker-compose/traefik_creds
htpasswd -nb admin  "$(secrethub read ${SECRETHUB_USERNAME}/docker-compose/nginx_pass)" | secrethub write ${SECRETHUB_USERNAME}/docker-compose/nginx_creds
htpasswd -nb admin  "$(secrethub read ${SECRETHUB_USERNAME}/docker-compose/netdata_pass)" | secrethub write ${SECRETHUB_USERNAME}/docker-compose/netdata_creds
```

Para comprobar que se han creado correctamente los secretos:

```bash
secrethub read ${SECRETHUB_USERNAME}/docker-compose/traefik_pass
secrethub read ${SECRETHUB_USERNAME}/docker-compose/netdata_pass
secrethub read ${SECRETHUB_USERNAME}/docker-compose/nginx_pass

secrethub read ${SECRETHUB_USERNAME}/docker-compose/traefik_creds
secrethub read ${SECRETHUB_USERNAME}/docker-compose/netdata_creds
secrethub read ${SECRETHUB_USERNAME}/docker-compose/nginx_creds
```

#### Creación del resto de secretos

```bash
echo $EMAIL | secrethub write ${SECRETHUB_USERNAME}/docker-compose/my_email
echo $PILOT_TOKEN | secrethub write ${SECRETHUB_USERNAME}/docker-compose/traefik_pilot_token
echo $TRANSMISSION_USER | secrethub write ${SECRETHUB_USERNAME}/docker-compose/transmission_user
secrethub generate rand ${SECRETHUB_USERNAME}/docker-compose/transmission_pass
```

Para comprobar que se han creado correctamente los secretos:

```bash
secrethub read ${SECRETHUB_USERNAME}/docker-compose/my_email
secrethub read ${SECRETHUB_USERNAME}/docker-compose/traefik_pilot_token
secrethub read ${SECRETHUB_USERNAME}/docker-compose/transmission_user
secrethub read ${SECRETHUB_USERNAME}/docker-compose/transmission_user
```

### Creación certificados SSL

Para la creación de certificado de un nuevo servicio, la entrada DNS debe estar creada como tipo `A`. Una vez generados los certificados, ya se pueden configurar como `CNAME` apuntando al __FQDN__ correcto

### Sacar /var/lib/docker de microSD

```bash
rm -fr /var/lib/docker/*
mkdir /datos/docker
mount --rbind /datos/docker /var/lib/docker
```

## Despliegue

```bash
secrethub run --template secrets.env.tpl -- docker-compose up -d
```

## Destrucción del entorno

```bash
secrethub run --template secrets.env.tpl -- docker-compose down
```
