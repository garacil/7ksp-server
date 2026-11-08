# 7KSP/1 — Protocolo 7ks

Versión del protocolo: 1. Estado del documento: borrador 0.2.

Este documento define todo lo necesario para implementar un cliente o un nodo 7KSP interoperable. Está escrito para que la implementación completa de un nodo quepa en unos pocos miles de líneas en cualquier lenguaje, usando solo componentes estándar: HTTPS, WebSocket, JSON, Ed25519, X25519, WebRTC.

Las palabras clave DEBE, NO DEBE, DEBERÍA y PUEDE (MUST, MUST NOT, SHOULD y MAY) se interpretan según la RFC 2119.

---

## 1. Qué hace el protocolo

7KSP responde a una única pregunta: *dada una dirección de email, ¿cómo abro un canal directo, cifrado y autenticado con uno de los dispositivos de esa persona?*

Para ello define cuatro cosas:

1. **Identidad**: cómo demuestra un dispositivo que pertenece a una dirección de email.
2. **Directorio**: dónde se publican los dispositivos de una dirección de email y cómo se buscan.
3. **Encuentro**: cómo entrega un nodo una petición de conexión a un dispositivo y le ayuda a atravesar el NAT.
4. **Canal directo**: cómo comprueban los dos dispositivos que hablan entre sí y no con un intermediario.

No define qué se transmite por el canal. El chat, las llamadas, los ficheros o cualquier otra aplicación se negocian por nombre al conectar y funcionan por encima.

### 1.1 Actores

| Actor | Qué es |
|---|---|
| Usuario | Una dirección de email |
| Dispositivo | Una instalación del cliente, con sus propias claves |
| Nodo de dominio | Servidor autoritativo de uno o varios dominios, anunciado en el DNS del dominio |
| Nodo de shard | Servidor del directorio público, responsable de una parte del espacio de hashes |
| Federación | Conjunto de nodos de shard con una tabla firmada común |
| Verificador | Servicio que envía un enlace a una dirección de email y certifica que alguien lo abrió |
| Home | Nodo al que está conectado un dispositivo en ese momento y a través del cual se le localiza |
| Pasarela de push | Servicio que despierta un dispositivo móvil dormido |

Un mismo servidor PUEDE desempeñar varios papeles.

### 1.2 Principios

- **El cliente es la unidad de almacenamiento.** Ningún nodo almacena nunca contenido de aplicación. Lo que no se ha entregado espera en el dispositivo emisor.
- **Lo que un nodo guarda de un usuario cabe en una línea.** Sus entradas de dispositivo, si está conectado y quién espera a que se conecte.
- **El protocolo es neutral.** Ninguna federación ni implementación recibe un trato especial. Los dominios son soberanos sobre sus direcciones.
- **Todo lo público es verificable sin confiar en quien lo sirve.** Cada objeto del directorio lleva su firma.

---

## 2. Convenciones

### 2.1 Codificación

- Todos los mensajes y objetos son JSON (RFC 8259), UTF-8.
- Los bytes (claves, firmas, hashes) se codifican en **base64url sin relleno** (RFC 4648 §5).
- Los tiempos son enteros: segundos Unix, UTC.
- Los campos desconocidos DEBEN ignorarse. Los campos definidos como obligatorios DEBEN estar presentes.

### 2.2 Firmas

Todo objeto firmado tiene un campo `sig`. La firma se calcula así:

```
sig = base64url( Ed25519_sign( clave_privada, JCS( objeto sin el campo "sig" ) ) )
```

`JCS` es JSON Canonicalization Scheme (RFC 8785). Con él, todas las implementaciones producen los mismos bytes a firmar sin acordar un orden de campos.

Para verificar: quitar `sig`, canonicalizar y comprobar contra la clave pública indicada.

### 2.3 Criptografía

| Uso | Algoritmo |
|---|---|
| Firma | Ed25519 (RFC 8032) |
| Acuerdo de claves / cifrado hacia un dispositivo | X25519 + *sealed box* de libsodium (`crypto_box_seal`) |
| Hash | SHA-256 |
| Transporte hacia los nodos | TLS 1.3 (certificado web ordinario del nodo) |
| Canal directo | WebRTC: DTLS 1.2+ y SRTP |

No hay agilidad criptográfica en la versión 1: una implementación no tiene que negociar nada. Cambiar un algoritmo será la versión 2.

### 2.4 Identificadores

| Identificador | Definición |
|---|---|
| `uid` | `sha256( normalize(email) )`, 32 bytes, hexadecimal en minúsculas en una URL y base64url en JSON |
| `dev_pk` | Clave pública Ed25519 del dispositivo, 32 bytes |
| `dev_dh` | Clave pública X25519 del dispositivo, 32 bytes |
| `nid` | `sha256( node_pk )`, donde `node_pk` es la clave pública Ed25519 del nodo |
| `dom_pk` | Clave pública Ed25519 del dominio |
| `fed_pk` | Clave pública Ed25519 de la federación |

`normalize(email)`: forma Unicode NFC, minúsculas, sin espacios al principio ni al final. **Ninguna otra transformación** (no se eliminan los puntos ni los sufijos `+`). El `uid` no tiene prefijo ni sal: cualquier sistema PUEDE calcularlo conociendo solo la dirección de email.

Vectores de prueba:

```
alice@example.com    → ff8d9819fc0e12bf0d24892e45987e249a28dce836a85cad60e28eaaa8c6d976
Alice@Example.COM    → ff8d9819fc0e12bf0d24892e45987e249a28dce836a85cad60e28eaaa8c6d976  (mismo uid)
garacil@7kas.com     → bfc98f4ae31a81b48d7865e69bf9fa3d657f23a80bb2af3c6aa42609032cec3b
```

El `prefijo` de un `uid` son sus dos primeros bytes como entero de 16 bits (`0xff8d` = 65421 en el primer vector).

---

## 3. Objetos

### 3.1 Prueba de titularidad

Certifica que `dev_pk` pertenece a `uid`. Tiene dos formas, según quién la emita.

Emitida por un verificador (directorio público):

```json
{
  "type": "attestation",
  "uid": "...", "dev_pk": "...",
  "verifier_pk": "...",
  "iat": 1762560000, "exp": 1765152000,
  "sig": "..."
}
```

Emitida por el dominio (nodo de dominio):

```json
{
  "type": "domainsig",
  "uid": "...", "dev_pk": "...",
  "domain": "7kas.com",
  "iat": 1762560000, "exp": 1765152000,
  "sig": "..."
}
```

Reglas: `exp - iat` ≤ 30 días. Una prueba caducada no vale nada.

### 3.2 Entrada de dispositivo

La unidad del directorio. La publica el propio dispositivo.

```json
{
  "type": "device",
  "v": 1,
  "uid": "...",
  "dev_pk": "...",
  "dev_dh": "...",
  "name": "Laptop",
  "homes": ["https://node.7kas.com"],
  "apps": ["7ks-chat/1", "7ks-call/1", "7ks-file/1"],
  "seq": 3,
  "exp": 1765152000,
  "proof": { "type": "domainsig", "...": "..." },
  "sig": "..."
}
```

| Campo | Significado |
|---|---|
| `homes` | URL base HTTPS de los nodos a través de los cuales se localiza este dispositivo, por orden de preferencia |
| `apps` | Protocolos de aplicación que acepta; nombres de la forma `nombre/versión` |
| `seq` | Estrictamente creciente por `dev_pk`. Un nodo NO DEBE aceptar un `seq` ≤ al almacenado |
| `exp` | NO DEBE superar `proof.exp` |
| `sig` | Firma con la clave privada de `dev_pk` |

Un nodo que recibe una entrada DEBE verificar: `sig` contra `dev_pk`; `proof.sig` contra el verificador o la `dom_pk` correspondiente; que `proof.uid` y `proof.dev_pk` coinciden con los de la entrada; `seq` y `exp`.

### 3.3 Registro de identidad

El conjunto de entradas vigentes de un `uid`. No tiene firma propia: cada entrada se verifica por separado.

```json
{ "type": "record", "uid": "...", "devices": [ { "type": "device", "...": "..." } ] }
```

Un nodo DEBE omitir las entradas caducadas al servirlo.

### 3.4 Registro de nodo

```json
{
  "type": "node",
  "v": 1,
  "nid": "...", "node_pk": "...",
  "url": "https://node.7kas.com",
  "roles": { "domains": ["7kas.com"], "shards": [], "verifier": false },
  "seq": 12, "exp": 1765152000,
  "sig": "..."
}
```

`url` es la base de toda la API del nodo (§8). Un nodo tiene una única URL base; la redundancia se consigue con DNS y balanceadores de carga, fuera del protocolo.

### 3.5 Tabla de shards

```json
{
  "type": "shards",
  "v": 1,
  "fed_id": "7ks",
  "nodes": { "<nid>": "https://shard3.example", "...": "..." },
  "prefixes": [ ["<nid>", "<nid>", "<nid>"], "... 65536 entradas ..." ],
  "verifiers": ["<verifier_pk>", "..."],
  "seq": 40, "exp": 1765152000,
  "prev_sig": "...",
  "sig": "..."
}
```

- `prefixes[i]` es la lista de nodos responsables del prefijo `i`, de 0 a 65535. Cada lista DEBERÍA tener al menos 3 nodos de operadores distintos.
- `verifiers` son las claves cuyas atestaciones acepta la federación.
- `sig` es de `fed_pk`. `prev_sig`, presente durante 90 días tras una rotación de clave, es la firma con la `fed_pk` anterior.
- Tamaño orientativo: 65536 × 3 referencias ≈ 2 MB en JSON. Un cliente la descarga una vez y la refresca por `seq`.

### 3.6 Vigilancia (watch)

Lo único que un nodo almacena de forma persistente sobre el tráfico. Vive en el nodo del destinatario.

```json
{ "watcher_uid": "...", "watcher_home": "https://nodeA.example", "target_uid": "...", "exp": 1763164800 }
```

---

## 4. Dominios

Un dominio se une a la red publicando en su DNS:

```
_7ks.<dominio>.   TXT   "v=7ksp1; url=https://node.<dominio>; pk=<dom_pk en base64url>"
```

Alternativa para quien no puede tocar el DNS con comodidad: `https://<dominio>/.well-known/7ks` devolviendo

```json
{ "v": "7ksp1", "url": "https://node.example.com", "pk": "<dom_pk>" }
```

Si existen ambos, gana el TXT. Un cliente DEBERÍA cachear el resultado respetando el TTL del DNS y en ningún caso durante más de 24 horas.

Un dominio con esta entrada es **autoritativo**:

- Solo su nodo publica entradas de sus usuarios, siempre con `domainsig`.
- Un nodo de shard NO DEBE servir entradas de un dominio con una entrada DNS vigente.
- Un cliente NO DEBE aceptar, para ese dominio, entradas con `attestation`.
- Si el nodo del dominio responde `404` para un `uid`, ese usuario no existe. El cliente NO DEBE consultar el directorio público.

Un nodo PUEDE servir varios dominios, cada uno con su propia `dom_pk`.

---

## 5. Federación

Una federación es: un `fed_id`, una `fed_pk`, una tabla de shards firmada y una URL de arranque desde la que descargarla.

El cliente se configura con una o varias federaciones. El protocolo no distingue entre ellas; el Anexo A describe la federación pública de 7ks, que es la predeterminada en el cliente 7ks.

Obligaciones de un nodo de shard:

- Mantener la tabla al día: descargarla de la URL de arranque o de cualquier otro nodo de shard y aceptarla solo si `sig` verifica contra `fed_pk` y `seq` es mayor.
- Replicar sus prefijos: como máximo cada 5 minutos, pedir a los demás nodos del mismo prefijo las entradas con un `seq` nuevo (§8.4) y aceptar las que verifiquen.
- Servir las lecturas públicas (§8.1) sin autenticación.

Obligaciones de un operador de federación (fuera del protocolo, pero necesarias): publicar los criterios de admisión de nodos, mantener la cobertura de los 65536 prefijos y rotar `fed_pk` con `prev_sig`.

---

## 6. Resolución

Entrada: una dirección de email. Salida: su registro de identidad verificado, o la certeza de que no existe.

```
1. Si hay un registro en caché local con entradas vigentes → usarlo.
2. dominio = parte tras la '@'. Consultar _7ks.<dominio> TXT, o /.well-known/7ks.
   2a. Existe:
       GET <url>/7ks/v1/record/<uid hex>
       200 → verificar que cada entrada lleva una domainsig válida contra pk. Devolver.
       404 → no existe. FIN.
3. Sin entrada de dominio:
   prefijo = uid[0..2]
   Elegir un nid de prefixes[prefijo]; GET <nodes[nid]>/7ks/v1/record/<uid hex>
   200 → verificar que cada entrada lleva una attestation de una verifier_pk de verifiers. Devolver.
   404 → probar otro nid del prefijo; si todos dicen 404 → no existe. FIN.
   307 → seguir Location una vez.
4. Fijación (TOFU):
   - Primera vez: guardar el conjunto de dev_pk y, si hay dominio, la dom_pk.
   - Dispositivo nuevo con prueba válida: añadirlo y avisar a la aplicación ("nuevo dispositivo").
   - dom_pk distinta de la guardada: avisar de posible suplantación; no usar sin confirmación.
```

Un cliente PUEDE delegar los pasos 2 y 3 en su home con `resolve` (§8.3), que hace lo mismo en su nombre. El resultado DEBE verificarse igualmente en el cliente.

---

## 7. Alta y vinculación de dispositivos

Un único procedimiento da de alta a un usuario nuevo, añade un dispositivo a un usuario existente y recupera la cuenta tras perder todos los dispositivos. La prueba es siempre la misma: abrir un enlace enviado a la dirección de email.

### 7.1 Flujo

```
Dispositivo                                  Verificador                    Buzón del usuario
    |-- POST /7ks/v1/verify {email, dev_pk, dev_dh} -->|
    |<-- 202 {req_id} ------------------------------|
    |                                               |-- enlace --------->|
    |-- GET /7ks/v1/verify/<req_id>  (long-poll) -->|                    |
    |                                               |<-- usuario abre -- |
    |<-- 200 {proof} -------------------------------|
    |-- construye y firma la entrada de dispositivo
    |-- PUT /7ks/v1/record/<uid>/<dev_pk> {entrada} --> home / nodo de dominio / nodos del prefijo
```

- El verificador es el nodo de dominio si el dominio tiene entrada DNS; si no, cualquier nodo con `roles.verifier` cuya clave esté en los `verifiers` de la federación.
- El email contiene dos enlaces con el mismo token: `https://<verificador>/7ks/v1/verify/<token>/confirm` para abrir en cualquier navegador y `7ks:verify?token=<token>` para abrir directamente en la aplicación.
- El `token` DEBE ser aleatorio, de al menos 128 bits, y caducar a los 15 minutos. `req_id` y `token` DEBEN ser valores distintos: `req_id` lo conoce el dispositivo; `token`, solo quien lee el email.
- `GET /verify/<req_id>` responde `200 {proof}` cuando el token se ha usado, `202` mientras espera (el servidor PUEDE retener la respuesta hasta 30 s) y `410` si caducó.

### 7.2 Obligaciones del verificador

- Enviar el enlace solo a la dirección indicada.
- Emitir la prueba solo después de que se haya abierto el token.
- Limitar la frecuencia: DEBERÍA rechazar más de 5 peticiones por dirección y hora, y aplicar límites por origen de red.
- No conservar el email tras emitir la prueba; conservar solo el `uid`.

### 7.3 Desvinculación

```
DELETE /7ks/v1/record/<uid>/<dev_pk>
body: { "uid": "...", "dev_pk": "...", "by": "<dev_pk firmante>", "iat": ..., "sig": "..." }
```

`sig` DEBE ser de otro dispositivo vigente del mismo `uid`, del verificador (tras un nuevo enlace) o de `dom_pk`. El nodo DEBE borrar la entrada y notificar `record_changed` (§8.3) a los clientes conectados suscritos a ese `uid`.

### 7.4 Dominios

Un nodo de dominio PUEDE sustituir el flujo del enlace por lo que decida el dominio (login corporativo, alta manual), siempre que el resultado sea una `domainsig`.

---

## 8. API del nodo

Toda la API vive bajo `<url>/7ks/v1/`. Las rutas de lectura son HTTPS sin autenticar; las de escritura llevan objetos firmados; la sesión del dispositivo es un WebSocket.

### 8.1 Lecturas públicas

| Ruta | Respuesta |
|---|---|
| `GET /record/<uid hex>` | `200` registro de identidad · `404` · `307 Location` si el nodo no es responsable y sabe quién lo es |
| `GET /node` | `200` registro de nodo |
| `GET /shards` | `200` tabla de shards (solo nodos de shard) |
| `GET /shards?since=<seq>` | `200` tabla · `304` si no ha cambiado |

Estas rutas NO DEBEN exigir autenticación ni limitar por origen más allá de lo razonable contra abusos. Cada respuesta es verificable por sus firmas.

### 8.2 Escrituras

| Ruta | Cuerpo | Respuesta |
|---|---|---|
| `PUT /record/<uid hex>/<dev_pk>` | entrada de dispositivo | `204` · `400` entrada no válida · `409` seq no mayor · `403` nodo no responsable |
| `DELETE /record/<uid hex>/<dev_pk>` | revocación firmada (§7.3) | `204` · `403` |
| `POST /verify` | `{email, dev_pk, dev_dh}` | `202 {req_id}` · `429` |
| `GET /verify/<req_id>` | — | `200 {proof}` · `202` · `410` |
| `GET /verify/<token>/confirm` | — | página de confirmación; marca el token como usado |

Un nodo es responsable de un `uid` si: es el nodo de dominio de ese dominio, o es un nodo de shard del prefijo y el dominio no tiene entrada DNS. En cualquier otro caso responde `403` con `{ "responsible": ["<url>", "..."] }` si lo sabe.

### 8.3 Sesión del dispositivo (WebSocket)

Un dispositivo conectado a un nodo abre `wss://<url>/7ks/v1/session`. Mientras la sesión está abierta, el dispositivo está **en línea** en ese nodo. Cerrarla es desconectarse. No hay otro mecanismo de presencia.

Todos los mensajes son objetos JSON con `t` (tipo). Las peticiones llevan `id` (una cadena elegida por el emisor) y la respuesta la repite.

**Autenticación.** El nodo abre con un desafío; el dispositivo responde firmado.

```
← { "t": "challenge", "nonce": "<32 bytes>" }
→ { "t": "auth", "uid": "...", "dev_pk": "...", "nonce": "...", "host": "node.7kas.com",
    "push": { "gateway": "https://push.7ks.app", "token": "..." },      // opcional
    "sig": "<firma de {uid, dev_pk, nonce, host}>" }
← { "t": "ok", "time": 1762560000,
    "ice_servers": [ { "urls": "stun:node.7kas.com:3478" },
                     { "urls": "turn:node.7kas.com:3478", "username": "...", "credential": "..." } ] }
```

El nodo DEBE comprobar que `dev_pk` tiene una entrada vigente para `uid` (en su propio directorio si es responsable; resolviendo si no). `host` impide que una firma sea válida en otro nodo. `ice_servers` sigue el formato de WebRTC; las credenciales TURN DEBERÍAN caducar al cabo de 1 hora.

**Mensajes del dispositivo al nodo.**

| Mensaje | Respuesta | Función |
|---|---|---|
| `{t:"resolve", id, uid}` | `{t:"record", id, record}` · `{t:"error", id, code:"not_found"}` | El nodo ejecuta §6 en nombre del dispositivo |
| `{t:"connect", id, env}` | `{t:"sent", id, devices:[dev_pk]}` · `{t:"offline", id}` · `{t:"error", id, code:"unknown"}` | Entregar un sobre de señalización (§9) a los dispositivos en línea del destinatario |
| `{t:"watch", id, uid, ttl}` | `{t:"ok", id}` · `{t:"error", id, code:"limit"}` | Avisar cuando `uid` esté en línea (§11) |
| `{t:"unwatch", id, uid}` | `{t:"ok", id}` | |
| `{t:"subscribe", id, uids:[...]}` | `{t:"ok", id}` | Recibir `record_changed` de esos `uid` |
| `{t:"block", id, uid}` / `{t:"unblock", id, uid}` | `{t:"ok", id}` | El nodo descarta todo lo que llegue de `uid` a este usuario |
| `{t:"ping"}` | `{t:"pong"}` | Mantener viva la sesión; cada 30 s |

**Mensajes del nodo al dispositivo.**

| Mensaje | Función |
|---|---|
| `{t:"deliver", env}` | Sobre de señalización recibido |
| `{t:"online", uid}` | Un `uid` al que este dispositivo esperaba está en línea |
| `{t:"record_changed", uid}` | El registro de un `uid` suscrito ha cambiado; volver a resolver |

`error.code` para `connect`: `unknown` se devuelve tanto para usuarios inexistentes como bloqueados; son indistinguibles por diseño.

### 8.4 Entre nodos

Los nodos se comunican entre sí por HTTPS con los mismos objetos. Cada petición entre nodos lleva:

```
Authorization: 7ks-node <nid> <ts> <sig>
```

con `sig = Ed25519( node_pk, JCS({ "method": ..., "path": ..., "ts": ..., "body_sha256": ... }) )`. El receptor DEBE rechazar un `ts` que difiera en más de 5 minutos y DEBE conocer `node_pk` (de su registro de nodo, obtenido con `GET /node` en la `url` que corresponde a `nid` en la tabla de shards o en la entrada DNS del dominio).

| Ruta | Función |
|---|---|
| `POST /relay` `{env}` | Entregar un sobre a un dispositivo de este nodo. Responde como `connect` |
| `POST /watch` `{watch}` | Registrar una vigilancia en este nodo (es el nodo del destinatario) |
| `POST /online` `{uid, watcher_uid}` | Avisar a este nodo de que un destinatario de uno de sus usuarios está en línea |
| `GET /sync/<prefijo>?since=<ts>` | Entradas del prefijo modificadas desde `since` (solo entre nodos de shard del mismo prefijo) |

Un nodo que recibe un `connect` o un `watch` para un `uid` que no es suyo DEBE resolver (§6) y reenviar exactamente una vez a un home del destinatario. Un `POST /relay` recibido nunca se reenvía.

---

## 9. Señalización

### 9.1 Sobre

Lo que transporta el nodo. El nodo ve el destinatario, el origen y la caducidad; el contenido es opaco.

```json
{
  "id": "<uuid>",
  "to":   { "uid": "...", "dev": "<dev_pk> | *" },
  "from": { "uid": "...", "dev": "<dev_pk>" },
  "exp": 1762560120,
  "body": "<base64url>"
}
```

- `to.dev = "*"`: el nodo entrega a todos los dispositivos en línea del `uid`.
- `exp`: NO DEBE superar `ahora + 120 s`. Un nodo que no puede entregar antes de `exp` descarta el sobre. **Un nodo NO DEBE encolar sobres**: si el destinatario no está en línea en ese momento, responde `offline`.
- `body` ≤ 16 KB.

### 9.2 Cuerpo

```
plaintext = JCS({ "signal": <Señal>, "from_dev": "<dev_pk>", "sig": "<firma Ed25519 del emisor sobre signal>" })
body      = sealed_box( dev_dh del destinatario, plaintext )
```

El receptor DEBE: abrir la sealed box con su clave X25519; comprobar que `from_dev` es una `dev_pk` vigente de `from.uid` (resolviendo si no la tiene); verificar `sig`. Si falla cualquier paso, descartar sin responder.

### 9.3 Señales

```json
{ "type": "offer",  "sid": "<128 bits>", "apps": ["7ks-call/1", "sip/2"], "sdp": "<oferta SDP>" }
{ "type": "answer", "sid": "...", "app": "7ks-call/1", "sdp": "<respuesta SDP>" }
{ "type": "reject", "sid": "...", "reason": "declined | busy | no_app | blocked" }
{ "type": "ice",    "sid": "...", "candidate": "<candidato ICE>", "mid": "..." }
{ "type": "bye",    "sid": "..." }
```

- `sid` lo genera el iniciador, al azar. Todas las señales de una sesión llevan el mismo `sid`; las que tienen un `sid` desconocido o terminado se descartan.
- `apps` es la lista de protocolos de aplicación que propone el iniciador, por orden de preferencia. El receptor elige uno. La aplicación del receptor decide si suena, acepta en silencio o rechaza, según la `app`; 7KSP solo entrega.
- `sdp` es una descripción SDP de WebRTC. Incluye la huella DTLS, que es lo que después vincula el canal a la identidad (§10).
- Una `offer` enviada con `to.dev = "*"` llega a varios dispositivos. Con la primera `answer`, el iniciador DEBE enviar `bye` a los demás.

### 9.4 Temporizadores

| Evento | Valor |
|---|---|
| Sin `answer` tras `offer` | 60 s → `bye` |
| Sin conexión ICE tras `answer` | 30 s → `bye` |
| Intervalo de reenvío de `ice` | según el agente ICE (trickle) |

---

## 10. Canal directo

El canal es una **PeerConnection de WebRTC** estándar. Se elige porque resuelve en una sola pieza, disponible en todos los lenguajes y navegadores, lo que de otro modo habría que especificar: ICE, DTLS, SRTP y canales de datos.

### 10.1 Establecimiento

1. El iniciador crea la PeerConnection con los `ice_servers` recibidos en su sesión, genera la oferta y la envía en una señal `offer`.
2. El receptor crea la suya, aplica la oferta, genera la respuesta y la envía en `answer`.
3. Ambos intercambian candidatos con `ice`.
4. ICE elige el par: directo (host o reflexivo) y, solo si no hay otro, TURN.

### 10.2 Vinculación a la identidad

La huella DTLS (`a=fingerprint` en el SDP) viajó dentro de un cuerpo firmado por `dev_pk` y cifrado para el destinatario. Cada lado DEBE comprobar, al completarse DTLS, que la huella del certificado remoto coincide con la del SDP recibido. Así, nadie puede interponerse sin la clave privada del dispositivo. No hay ningún otro intercambio de huellas ni de claves.

### 10.3 Uso

- La aplicación elegida (`app` en la `answer`) es la dueña de la conexión. Por convención, abre sus canales de datos con `protocol` igual al nombre de la aplicación (`createDataChannel("main", { protocol: "7ks-chat/1" })`), y las pistas de audio y vídeo son las de WebRTC.
- 7KSP termina aquí. Nada de lo que va por el canal vuelve a tocar un nodo.
- Si la conexión cae, la aplicación PUEDE intentar reconectar con una nueva `offer` con el mismo `sid` (reinicio ICE) durante 60 s; después, con un `sid` nuevo.

### 10.4 Transportes alternativos

Una implementación nativa PUEDE ofrecer QUIC directo como alternativa a WebRTC, anunciándolo en `apps` con el sufijo `+quic` (`7ks-file/1+quic`). Es opcional y no afecta a la interoperabilidad: todo cliente DEBE soportar WebRTC.

---

## 11. Entrega diferida

Sustituye a cualquier cola de mensajes. El nodo no almacena contenido; almacena quién espera a quién.

Cuando el dispositivo A quiere entregar algo a B y B no está en línea:

```
1. A envía connect → recibe offline.
2. A guarda el contenido en su propio almacén y envía watch {uid: B, ttl}.
   El home de A hace POST /watch en el home de B (resolviendo B).
3. B abre una sesión en su home. El home recorre las vigilancias con target_uid = B y, para cada una:
   - si el vigilante está conectado a este mismo nodo → le envía online;
   - si no → POST /online a watcher_home, que lo entrega por la sesión o lo convierte en un push silencioso.
   La vigilancia se borra al emitirse.
4. A recibe online, envía connect y entrega por el canal directo.
5. Si A no recibió el aviso (estaba apagado y el push no lo despertó), reintenta al arrancar.
```

Reglas:

- Un cliente DEBE reintentar todo lo pendiente al arrancar y al recuperar la conectividad.
- Un nodo DEBE aceptar como máximo 256 vigilancias activas por `watcher_uid` y `ttl` ≤ 7 días. Una vigilancia se renueva enviándola de nuevo.
- `online` no revela nada que el vigilante no descubriría con un `connect`; aun así, un usuario PUEDE bloquear a otro y el nodo descartará sus vigilancias.

---

## 12. Push

Un dispositivo móvil dormido no tiene sesión abierta. Para alcanzarlo, el dispositivo registra en `auth` un `push` con la URL de una pasarela y un token opaco. La pasarela la elige el dispositivo, no el nodo.

Cuando un nodo necesita despertar a un dispositivo registrado sin sesión:

```
POST <pasarela>/7ks/v1/push
{ "token": "...", "kind": "connect" | "online" }
```

- `connect`: prioridad alta (push VoIP en iOS, prioridad alta en FCM). Lo dispara un sobre con una `offer` dirigido a ese dispositivo. El nodo DEBE retener el sobre hasta `exp` (120 s como máximo) mientras espera a que el dispositivo abra sesión; es la única retención de sobres permitida.
- `online`: prioridad baja, silencioso; puede no entregarse. §11.5 lo cubre.

La pasarela no recibe ningún `uid`, `sid` ni contenido. Cualquiera PUEDE operar una pasarela para su propia compilación del cliente. UnifiedPush es un transporte válido en Android sin la pasarela de nadie: en ese caso `push.gateway` es el endpoint UnifiedPush del dispositivo.

---

## 13. Errores

Las respuestas HTTP usan códigos estándar y un cuerpo `{ "error": "<código>", "detail": "..." }`. Los mensajes de sesión usan `{ "t": "error", "id": ..., "code": "<código>" }`.

| code | Significado |
|---|---|
| `invalid` | Objeto mal formado o firma no válida |
| `expired` | Prueba, entrada o sobre caducado |
| `not_found` | El `uid` no existe en este nodo |
| `not_responsible` | El nodo no es responsable de ese `uid` |
| `seq` | `seq` no es mayor que el almacenado |
| `unknown` | Destinatario desconocido o bloqueado |
| `offline` | Ningún dispositivo del destinatario tiene sesión abierta |
| `limit` | Cuota superada |
| `unauthorized` | Autenticación de sesión o de nodo no válida |

---

## 14. Límites

| Parámetro | Valor |
|---|---|
| Sobre: `exp` máximo / tamaño | 120 s / 16 KB |
| Entrada de dispositivo: `exp` máximo | 30 días; renovar a los 20 |
| Dispositivos por `uid` | 32 |
| Token de verificación | 15 minutos |
| Peticiones de verificación | 5 por dirección y hora |
| Vigilancias por vigilante / `ttl` | 256 / 7 días |
| Credenciales TURN | 1 hora |
| TURN por sesión | cuota del operador; recomendado 1 Mbit/s y 10 minutos |
| Ping de sesión | 30 s; el nodo cierra tras 90 s sin tráfico |
| Tabla de shards: `exp` | 7 días |

---

## 15. Seguridad

**Modelo de identidad.** Controlar el email es controlar la cuenta. Es una decisión deliberada: es lo que permite entrar desde cualquier dispositivo solo con el email. Las consecuencias se mitigan en el cliente, no en el protocolo: un aviso al usuario y a sus contactos cuando aparece un dispositivo nuevo, y la posibilidad de revocarlo desde cualquier otro.

**Qué ve un nodo.** El `uid` de origen y de destino de cada sobre que pasa por él, las vigilancias y qué dispositivos tienen sesión abierta. No ve el contenido de la señalización ni de la aplicación, ni puede alterarlos. Quien no quiera que un tercero vea ni siquiera eso, opera un nodo de dominio.

**Qué puede hacer un nodo malicioso.** Denegar el servicio a sus usuarios, responder falsamente `offline` o `unknown` y retener sobres. No puede leer, modificar ni suplantar, porque todo va firmado con claves que nunca salen de los dispositivos.

**Suplantación de direcciones.** En un dominio con entrada DNS requiere la clave privada del dominio. En el directorio público requiere engañar a un verificador aceptado por la federación. La fijación en el cliente convierte un cambio de `dom_pk` en una alerta, y un dispositivo nuevo en un aviso.

**Canal directo.** La huella DTLS queda vinculada a `dev_pk` por la firma del cuerpo de la señal. Un intermediario tendría que falsificar esa firma.

**Enumeración.** El `uid` es un hash sin sal, así que quien tenga una lista de direcciones de email puede comprobar cuáles están en el directorio público. Se acepta: el directorio no devuelve emails, solo claves, y los dominios con nodo propio no aparecen en él.

**Spam de conexiones.** Una `offer` solo llega a un dispositivo con sesión abierta y la aplicación decide si molesta al usuario. `block` corta el resto. Los nodos DEBERÍAN limitar la frecuencia de `connect` por origen.

---

## 16. Identificadores registrados

| Elemento | Valor |
|---|---|
| Registro DNS | `_7ks.<dominio>` TXT |
| Well-known | `/.well-known/7ks` |
| Prefijo de la API | `/7ks/v1/` |
| Esquema URI | `7ks:` — `7ks:alice@example.com`, `7ks:alice@example.com?app=7ks-call/1`, `7ks:verify?token=…` |
| Nombres de aplicación | `nombre/versión`, en minúsculas, sin espacios; `7ks-` reservado para las aplicaciones del cliente 7ks |

---

## Anexo A. Federación pública de 7ks (informativo)

| Campo | Valor |
|---|---|
| `fed_id` | `7ks` |
| `fed_pk` | pendiente de publicar |
| URL de arranque | pendiente de publicar |
| Verificadores iniciales | pendientes de publicar |

El operador de la federación pública publica, fuera de este documento: los criterios de admisión de nodos de shard, el compromiso de neutralidad, el plazo de restauración de réplicas y el procedimiento de rotación de `fed_pk`.

## Anexo B. Ejemplo completo: A llama a B

A es `garacil@7kas.com`, dispositivo A1, conectado al nodo de dominio N (`https://node.7kas.com`). B es `soporte@7kas.com`, dispositivos B1 y B2, mismo nodo.

```
A1 → N   ws  {t:"resolve", id:"1", uid:<B>}
N  → A1  ws  {t:"record", id:"1", record:{devices:[B1, B2]}}        // A1 verifica la domainsig y fija las claves
A1       crea la PeerConnection, genera la oferta SDP
A1 → N   ws  {t:"connect", id:"2", env:{to:{uid:<B>, dev:"*"}, from:{uid:<A>, dev:A1}, exp, body:sealed(B1, offer)}}
A1 → N   ws  {t:"connect", id:"3", env:{to:{uid:<B>, dev:B2}, ..., body:sealed(B2, offer)}}
N  → B1  ws  {t:"deliver", env}          // B1 tiene sesión
N  → B2  push {kind:"connect"}           // B2 está dormido; N retiene el sobre hasta exp
B2 → N   ws  auth…  → N → B2 {t:"deliver", env}
B1       abre el cuerpo, verifica la firma de A1, la aplicación 7ks-call/1 suena
B1 → N   ws  {t:"connect", env:{to:{uid:<A>, dev:A1}, body:sealed(A1, answer{app:"7ks-call/1"})}}
N  → A1  ws  {t:"deliver", env}
A1 → N   ws  {t:"connect", env:{to:{uid:<B>, dev:B2}, body:sealed(B2, bye)}}   // B2 deja de sonar
A1 ↔ B1  señales ice a través de N; ICE conecta; DTLS; huellas comprobadas contra los SDP
A1 ↔ B1  audio y vídeo directos. N ya no interviene.
```
