# Template Zabbix para UniFi Network Application / UniFi OS

Template para Zabbix 7 orientado al monitoreo de Access Points UniFi mediante la API local de UniFi.

El template soporta tanto instalaciones **self-hosted de UniFi Network Application** como equipos basados en **UniFi OS**, por ejemplo **UDM Pro**, detectando automáticamente el tipo de plataforma.

En instalaciones self-hosted también soporta múltiples sitios desde un mismo controller. Los sitios y Access Points se descubren automáticamente.

Probado con:

- Zabbix 7.0.x
- UniFi Network Application 10.x
- UniFi Network Application self-hosted
- UniFi Controller (Legacy)
- UniFi OS / UDM Pro
- Zabbix Server y Zabbix Proxy

## Funcionalidades

- Detección automática entre UniFi Network Application y UniFi OS
- Descubrimiento multi-site en controllers self-hosted
- Descubrimiento automático de Access Points
- Una única consulta principal a la API por ciclo
- Nombres de ítems y triggers identificados por sitio y AP
- Tags por sitio y Access Point para facilitar filtros en Zabbix
- No requiere SNMP ni Zabbix Agent en los AP

Actualmente el template obtiene:

- Estado del AP
- Dirección IP
- Modelo
- Versión de firmware
- Uptime
- Clientes conectados
- WiFi satisfaction
- Canal de 2.4 GHz
- Utilización de canal de 2.4 GHz
- Potencia TX de 2.4 GHz
- Retransmisiones TX de 2.4 GHz
- Clientes en 2.4 GHz
- Canal de 5 GHz
- Utilización de canal de 5 GHz
- Potencia TX de 5 GHz
- Retransmisiones TX de 5 GHz
- Clientes en 5 GHz

Triggers incluidos:

- AP offline durante 5 minutos
- Utilización alta del canal de 2.4 GHz
- Utilización alta del canal de 5 GHz
- Retransmisiones TX altas en 2.4 GHz
- Retransmisiones TX altas en 5 GHz
- WiFi satisfaction baja
- AP reiniciado recientemente

## Requisitos

- Zabbix 7.0 o superior
- UniFi Network Application self-hosted o UniFi OS compatible
- Usuario local de UniFi con permisos de solo lectura
- Conectividad HTTPS desde el Zabbix Server o Zabbix Proxy hacia el controller o gateway UniFi

No es necesario configurar una interfaz de host en Zabbix. El template utiliza un ítem de tipo Script y consulta directamente la API de UniFi.

## Instalación

1. Descargar `template_unifi_network_application.yaml`.
2. En Zabbix ir a **Data collection → Templates → Import**.
3. Importar el template.
4. Crear un host para el controller UniFi o UDM.
5. Asociar el template **UniFi Network Application by API**.
6. Si las consultas deben realizarse desde un Zabbix Proxy, asignar el host al proxy correspondiente.
7. Configurar las macros requeridas.
8. No agregar interfaces Agent o SNMP salvo que sean necesarias para otro tipo de monitoreo.

## Usuario de UniFi

Se recomienda crear un usuario local dedicado exclusivamente al monitoreo.

Los permisos de solo lectura son suficientes.

En instalaciones self-hosted con múltiples sitios, el usuario debe tener acceso a todos los sitios que se quieran descubrir desde Zabbix.

### UniFi Network Application self-hosted

El template utiliza:

```text
/api/login
/api/self/sites
/api/s/<site>/stat/device
```

`/api/self/sites` permite descubrir todos los sitios visibles para el usuario de monitoreo. Luego se consulta cada sitio y se descubren los dispositivos con tipo `uap`.

### UniFi OS / UDM Pro

En UniFi OS el template utiliza:

```text
/api/auth/login
/proxy/network/api/s/default/stat/device
```

El tipo de plataforma se detecta automáticamente. No es necesario indicar manualmente si el equipo es un UDM o un controller tradicional.

## Macros de Zabbix

Configurar las siguientes macros en el host:

| Macro | Ejemplo | Descripción |
| --- | --- | --- |
| `{$UNIFI.URL}` | `https://192.168.1.10:8443` | URL base del controller UniFi, sin `/` al final |
| `{$UNIFI.USER}` | `zabbix-monitor` | Usuario local utilizado para monitoreo |
| `{$UNIFI.PASSWORD}` | `********` | Contraseña del usuario de monitoreo |

`{$UNIFI.PASSWORD}` está definido como macro de tipo **Secret text**.

No existe una macro de sitio. Los sitios disponibles se descubren automáticamente cuando el controller lo soporta.

## Descubrimiento multi-site

En UniFi Network Application self-hosted, el template agrega a cada AP descubierto el ID y nombre del sitio al que pertenece.

Los ítems quedan identificados de esta forma:

```text
[Casa Central] AP AP-01: Status
[Sucursal] AP AP-02: Clients
```

Los triggers también incluyen el sitio:

```text
UniFi [Sucursal] AP AP-02: Offline for 5 minutes
```

Los ítems y triggers descubiertos incluyen tags con el sitio y nombre del AP. Los triggers también incluyen la dirección MAC.

Esto permite filtrar fácilmente Problems, dashboards y vistas dentro de Zabbix.

## Cómo funciona

El ítem principal de tipo Script se ejecuta una vez por minuto.

En cada ejecución:

1. Intenta autenticarse utilizando la API clásica de UniFi Network Application.
2. Si la autenticación es exitosa, obtiene todos los sitios visibles para el usuario.
3. Consulta los dispositivos de cada sitio.
4. Si el primer método no aplica, intenta autenticarse mediante UniFi OS.
5. En UniFi OS consulta los dispositivos de Network mediante `/proxy/network/api/...`.
6. Conserva únicamente los dispositivos de tipo `uap`.
7. Devuelve un único JSON con todos los Access Points encontrados.

Una regla de Low-Level Discovery procesa ese JSON y genera automáticamente los ítems dependientes para cada AP.

De esta forma no se realiza una consulta independiente a la API por cada métrica.

## Notas

### Zabbix Proxy

Cuando el host está asignado a un Zabbix Proxy, las consultas a la API se ejecutan desde ese proxy.

Por lo tanto, el proxy debe tener conectividad hacia la dirección configurada en `{$UNIFI.URL}`.

Por ejemplo:

```bash
curl -k https://192.168.1.10:8443/
```

En UDM o UniFi OS normalmente se utiliza HTTPS sobre el puerto 443, por ejemplo:

```text
https://192.168.1.1
```

### Permisos de sitios

Si un sitio no aparece en Zabbix, revisar primero los permisos del usuario local utilizado para monitoreo.

En controllers multi-site, solamente se podrán descubrir los sitios visibles para ese usuario.

### WiFi satisfaction

Algunos AP pueden devolver `-1` cuando la métrica de satisfaction no está disponible.

El trigger incluido ignora los valores negativos para evitar alertas incorrectas.

### Datos de radios

Actualmente el template interpreta:

```text
ng = 2.4 GHz
na = 5 GHz
```

Se utilizan los datos disponibles dentro de `radio_table_stats`.

Las métricas de 6 GHz todavía no están incluidas.

## Alcance actual

Actualmente el template está enfocado en Access Points UniFi.

No descubre ni monitorea todavía:

- Switches UniFi
- Gateways UniFi
- Clientes
- Estadísticas WLAN/SSID
- Radios de 6 GHz

Estas funciones pueden incorporarse en futuras versiones.

## Compatibilidad

El template fue desarrollado y probado con:

- Zabbix 7.0.30
- UniFi Network Application 10.3.58
- Controllers UniFi Network Application de versiones anteriores
- UniFi OS / UDM Pro

La API local de UniFi puede sufrir cambios entre versiones, por lo que se recomienda validar el funcionamiento luego de actualizaciones mayores.

## Seguridad

Se recomienda utilizar siempre un usuario local dedicado y de solo lectura para el monitoreo.

No incluir credenciales reales dentro del template ni publicar configuraciones de hosts que contengan datos sensibles.

El template no incluye:

- Direcciones IP de clientes
- IDs de sitios reales
- Nombres de clientes
- Usuarios o contraseñas reales

## Licencia

Este proyecto se distribuye bajo licencia MIT.