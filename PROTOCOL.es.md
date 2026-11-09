# 7KSP/1 — Protocolo 7ks

Versión del protocolo: 1. Estado del documento: borrador 0.6 (borrador, sin publicar).

7KSP es un protocolo abierto y libre que, a partir de una dirección de email, abre un canal directo, cifrado y autenticado entre dos dispositivos. Este documento contiene todo lo necesario para implementar un cliente o un nodo interoperable sin consultar ninguna otra fuente. Está escrito para que una implementación completa quepa en unos pocos miles de líneas en cualquier lenguaje, usando solo componentes estándar: HTTPS y WebSocket hacia los nodos, JSON, Ed25519, X25519, el framework Noise, STUN, UDP y, cuando interviene un navegador, WebRTC.

---

## 0. Cómo leer este documento

### 0.1 Palabras clave

DEBE, NO DEBE, DEBERÍA, NO DEBERÍA y PUEDE (MUST, MUST NOT, SHOULD, SHOULD NOT y MAY) se interpretan según la RFC 2119. Una implementación que incumple un DEBE no es conforme.

### 0.2 Niveles de conformidad

Cada implementación declara uno o varios de estos papeles. La tabla indica qué secciones son obligatorias para cada uno.

| Papel | Qué es | Secciones obligatorias |
|---|---|---|
| Cliente nativo | Aplicación de escritorio o móvil con sockets UDP | 2–4, 7, 8, 9.3, 10, 11 (perfil `noise`; `webrtc` recomendado), 12, 13 |
| Cliente web | Aplicación que se ejecuta en un navegador | 2–4, 7, 8, 9.3, 10, 11 (perfil `webrtc`), 12, 13 |
| Nodo de dominio | Servidor autoritativo de uno o varios dominios | 2–5, 8, 9, 11.2 (STUN), 12, 13, 14 |
| Nodo de shard | Servidor del directorio público | 2–4, 6, 9.1, 9.2, 9.4, 11.2 (STUN), 14 |
| Verificador | Servicio de enlace por email | 2–4, 8 |
| Relay | Servidor TURN anunciado en la red | 11.2 |
| Pasarela de push | Servicio que despierta dispositivos móviles | 13 |

Un mismo servidor PUEDE desempeñar varios papeles de servidor; lo habitual es que un nodo de dominio sea también el verificador y el relay de sus usuarios.

### 0.3 Glosario

| Término | Definición |
|---|---|
| Usuario | Una dirección de email. No tiene claves propias. |
| Dispositivo | Una instalación del cliente, con sus propias claves. Un usuario tiene uno o varios. |
| `uid` | Identificador de usuario: SHA-256 del email normalizado. |
| Entrada de dispositivo | Objeto firmado que publica un dispositivo: sus claves, cómo localizarlo y qué soporta. |
| Registro de identidad | Conjunto de entradas de dispositivo vigentes de un `uid`. |
| Prueba de titularidad | Objeto firmado por un verificador o por un dominio que vincula la clave de un dispositivo a un `uid`. |
| Nodo | Servidor que habla 7KSP. Tiene una URL base HTTPS y una clave Ed25519. |
| Home | Nodo al que está conectado un dispositivo en ese momento y a través del cual le llega la señalización. |
| Nodo de dominio | Nodo autoritativo para las direcciones de uno o varios dominios, anunciado en el DNS del dominio. |
| Nodo de shard | Nodo del directorio público, responsable de una parte del espacio de `uid`. |
| Shard, prefijo | Partición del directorio público: los dos primeros bytes del `uid`, de 0 a 65535. |
| Federación | Conjunto de nodos de shard que comparten una tabla firmada por una clave de federación. |
| Verificador | Servicio que envía un enlace a un email y, si alguien lo abre, emite una prueba de titularidad. |
| Encuentro | Entregar una petición de conexión a un dispositivo, con los datos necesarios para encontrarse a través del NAT. |
| Sobre | Mensaje de señalización que transporta un nodo: destinatario, origen, caducidad y un cuerpo opaco. |
| Señal | Contenido del sobre, cifrado para el destinatario y firmado por el origen: oferta, respuesta, candidatos, bye. |
| Canal directo | Conexión UDP entre dos dispositivos, cifrada y autenticada por sus claves, sin ningún nodo en el camino. |
| Perfil de transporte | Forma concreta del canal directo: `noise` o `webrtc`. Mismas garantías, distintos bytes en la red. |
| Relay | Servidor TURN que reenvía paquetes cifrados cuando dos dispositivos no pueden alcanzarse directamente. |
| Vigilancia (watch) | Aviso de interés: "avísame cuando este `uid` esté en línea". Lo único que un nodo almacena sobre el tráfico. |
| App | Protocolo de aplicación que funciona sobre el canal directo, identificado por `nombre/versión`. |

---

## 1. Visión general

### 1.1 Qué hace el protocolo

7KSP responde a una única pregunta: *dada una dirección de email, ¿cómo abro un canal directo, cifrado y autenticado con uno de los dispositivos de esa persona?* Para ello define cuatro cosas:

1. **Identidad** (§8): cómo demuestra un dispositivo que pertenece a una dirección de email. La prueba es abrir un enlace enviado a esa dirección.
2. **Directorio** (§4–§7): dónde se publican los dispositivos de una dirección de email y cómo se buscan. Un dominio con nodo propio es la autoridad sobre sus direcciones; el resto va a un directorio público repartido por el hash del email.
3. **Encuentro** (§9–§10): cómo entrega un nodo una petición de conexión a un dispositivo y cómo se encuentran los dos dispositivos a través del NAT.
4. **Canal directo** (§11): una conexión UDP entre los dos dispositivos, cifrada y autenticada por sus claves, sin autoridad de certificación y sin ningún nodo en el camino. Existe en dos perfiles, `noise` y `webrtc`, con las mismas garantías; cuál se usa se negocia en cada sesión.

No define qué viaja por el canal. El chat, las llamadas, los ficheros o cualquier otra aplicación se negocian por nombre al conectar y funcionan por encima.

### 1.2 El recorrido de una conexión en diez líneas

```
1. El dispositivo A1 (garacil@7kas.com) quiere llamar a soporte@7kas.com.
2. Resuelve el email: DNS de 7kas.com → nodo N → registro de identidad de B, firmado. Fija las claves (TOFU).
3. Prepara candidatos de red (su IP pública mediante STUN) y una oferta: qué app, qué transportes, cómo encontrarlo.
4. Cifra la oferta para cada dispositivo de B, la firma y la mete en un sobre.
5. Envía el sobre a su home. El home lo entrega a los dispositivos conectados de B; a los dormidos los despierta por push.
6. B1 recibe la oferta y la aplicación decide sonar. El usuario acepta.
7. B1 responde con sus candidatos, cifrados y firmados, por el mismo camino.
8. A1 y B1 se envían sondas UDP a la vez; sus NAT se abren; eligen un par de direcciones.
9. Handshake Noise (o DTLS): cada uno comprueba que el otro es exactamente la clave del directorio.
10. Audio como datagramas, señalización de la llamada por un stream. El nodo ya no interviene.
```

### 1.3 Actores

| Actor | Qué es |
|---|---|
| Usuario | Una dirección de email |
| Dispositivo | Una instalación del cliente, con sus propias claves |
| Nodo de dominio | Servidor autoritativo de uno o varios dominios, anunciado en el DNS del dominio |
| Nodo de shard | Servidor del directorio público, responsable de una parte del espacio de hashes |
| Federación | Conjunto de nodos de shard con una tabla firmada común |
| Verificador | Servicio que envía un enlace a un email y certifica que alguien lo abrió |
| Home | Nodo al que está conectado un dispositivo en ese momento y a través del cual se le localiza |
| Relay | Nodo que reenvía paquetes UDP cifrados entre dos dispositivos que no pueden alcanzarse directamente |
| Pasarela de push | Servicio que despierta un dispositivo móvil dormido |

### 1.4 Principios

- **El cliente es la unidad de almacenamiento.** Ningún nodo almacena nunca contenido de aplicación. Lo que no se ha entregado espera en el dispositivo emisor.
- **Lo que un nodo guarda de un usuario cabe en una línea.** Sus entradas de dispositivo, si está conectado y quién espera a que se conecte.
- **Siempre hay NAT.** Se asume que los dos dispositivos están detrás de NAT y no pueden aceptar conexiones entrantes. Todo el canal directo parte de ahí.
- **La confianza está en las claves, no en los certificados.** El directorio ya le dice a cada dispositivo las claves del otro. El canal directo no necesita autoridad de certificación ni nombre de dominio: funciona como SSH, no como HTTPS. Esto vale para los dos perfiles.
- **Un protocolo, varios transportes.** La identidad, el directorio, el encuentro, la entrega diferida y las primitivas que ve la aplicación son iguales en todas partes. Solo cambian los bytes en la red entre dos dispositivos, y la aplicación nunca lo nota.
- **El protocolo es neutral.** Ninguna federación ni implementación recibe un trato especial. Los dominios son soberanos sobre sus direcciones.
- **Todo lo público es verificable sin confiar en quien lo sirve.** Cada objeto del directorio lleva su firma.

---

## 2. Codificación y criptografía

### 2.1 Codificación

- Todos los mensajes y objetos que se intercambian con los nodos son JSON (RFC 8259) en UTF-8.
- Los bytes (claves, firmas, hashes) se codifican en **base64url sin relleno** (RFC 4648 §5): alfabeto `A–Z a–z 0–9 - _`, sin `=` final.
- Los tiempos son enteros: segundos Unix en UTC.
- Los campos desconocidos DEBEN ignorarse. Los campos definidos como obligatorios DEBEN estar presentes; su ausencia invalida el objeto.
- Los paquetes del perfil `noise` (§11.4) son binarios, con enteros en big-endian.

### 2.2 Firmas

Todo objeto firmado tiene un campo `sig`. La firma se calcula sobre la serialización canónica del objeto sin ese campo:

```
sign(objeto, clave_privada):
    o = copia del objeto sin el campo "sig"
    bytes = JCS(o)                       // RFC 8785: claves ordenadas, sin espacios, números canónicos
    return base64url( Ed25519_sign(clave_privada, bytes) )

verify(objeto, clave_publica):
    o = copia del objeto sin el campo "sig"
    return Ed25519_verify(clave_publica, JCS(o), base64url_decode(objeto.sig))
```

JCS (JSON Canonicalization Scheme) garantiza que todas las implementaciones producen exactamente los mismos bytes a firmar sin acordar un orden de campos ni un formato de números. Hay implementaciones en todos los lenguajes habituales; donde no la hay, son cien líneas.

### 2.3 Algoritmos

| Uso | Algoritmo | Referencia |
|---|---|---|
| Firma de objetos y señalización | Ed25519 | RFC 8032 |
| Cifrado de la señalización hacia un dispositivo | *sealed box* de libsodium: X25519 efímero + XSalsa20-Poly1305 | `crypto_box_seal` |
| Hash de identificadores | SHA-256 | FIPS 180-4 |
| Transporte hacia los nodos | HTTPS y WSS, TLS 1.3, certificado web ordinario del nodo | RFC 8446 |
| Canal directo, perfil `noise` | `Noise_IK_25519_ChaChaPoly_BLAKE2s` sobre UDP | Noise rev. 34 |
| Canal directo, perfil `webrtc` | DTLS 1.2 o posterior con certificado autofirmado; SRTP para medios | RFC 8827, RFC 8832 |

No hay agilidad criptográfica en la versión 1: una implementación no negocia nada. Cambiar un algoritmo será la versión 2 del protocolo.

### 2.4 Sealed box

Cifra un mensaje para una clave pública X25519 sin que el emisor necesite una clave propia de larga duración:

```
sealed_box(target_pk, mensaje):
    (epk, esk) = X25519_keypair()                       // efímera, se descarta después
    nonce = BLAKE2b-24( epk || target_pk )
    ct = XSalsa20-Poly1305( key = X25519(esk, target_pk), nonce, mensaje )
    return epk(32) || ct
```

El receptor recalcula el nonce con su propia clave pública y descifra con `X25519(target_sk, epk)`. Es exactamente `crypto_box_seal` de libsodium, disponible en todos los lenguajes; una implementación DEBERÍA usar la librería en lugar de reimplementarla.

---

## 3. Identificadores

| Identificador | Qué es | Tamaño |
|---|---|---|
| `uid` | `sha256( normalize(email) )` | 32 bytes; hexadecimal en minúsculas dentro de una URL, base64url dentro de JSON |
| `dev_pk` | Clave pública Ed25519 del dispositivo. Firma su entrada y su señalización | 32 bytes |
| `dev_dh` | Clave pública X25519 del dispositivo. Recibe la señalización sellada y es su identidad en el perfil `noise` | 32 bytes |
| `nid` | `sha256( node_pk )`, donde `node_pk` es la clave pública Ed25519 del nodo | 32 bytes |
| `dom_pk` | Clave pública Ed25519 del dominio, publicada en su DNS | 32 bytes |
| `fed_pk` | Clave pública Ed25519 de la federación | 32 bytes |

Las claves de un dispositivo se generan en el dispositivo y NO DEBEN salir de él. `dev_pk` y `dev_dh` son claves distintas, generadas de forma independiente; una no se deriva de la otra.

### 3.1 Normalización del email

```
normalize(email):
    1. Normalización Unicode NFC.
    2. Minúsculas (conversión completa de Unicode, no solo ASCII).
    3. Quitar los espacios al principio y al final.
    4. Nada más.
```

No se quitan los puntos, no se eliminan los sufijos `+etiqueta` y no se reescriben dominios: `juan.perez+7ks@gmail.com` y `juanperez@gmail.com` son dos `uid` distintos aunque Gmail los entregue en el mismo buzón. El motivo es que el protocolo no puede conocer las reglas de cada proveedor, y cualquier regla que aplicara rompería la promesa de que *cualquiera* calcula el mismo `uid`.

### 3.2 Vectores de prueba

```
alice@example.com    → ff8d9819fc0e12bf0d24892e45987e249a28dce836a85cad60e28eaaa8c6d976
Alice@Example.COM    → ff8d9819fc0e12bf0d24892e45987e249a28dce836a85cad60e28eaaa8c6d976  (mismo uid)
garacil@7kas.com     → bfc98f4ae31a81b48d7865e69bf9fa3d657f23a80bb2af3c6aa42609032cec3b
```

El `prefijo` de un `uid` son sus dos primeros bytes leídos como entero de 16 bits: `0xff8d` = 65421 en el primer vector; `0xbfc9` = 49097 en el tercero.

---

## 4. Objetos

Todo objeto lleva `type`. Los ejemplos usan `<…>` donde iría un valor real.

### 4.1 Prueba de titularidad

Certifica que `dev_pk` pertenece a `uid`. Tiene dos formas según quién la emita; las dos se verifican igual.

Emitida por un verificador, para direcciones del directorio público:

```json
{
  "type": "attestation",
  "uid": "<uid>",
  "dev_pk": "<dev_pk>",
  "verifier_pk": "<clave Ed25519 del verificador>",
  "iat": 1762560000,
  "exp": 1765152000,
  "sig": "<firma del verificador>"
}
```

Emitida por un dominio, para direcciones de un dominio con nodo propio:

```json
{
  "type": "domainsig",
  "uid": "<uid>",
  "dev_pk": "<dev_pk>",
  "domain": "7kas.com",
  "iat": 1762560000,
  "exp": 1765152000,
  "sig": "<firma con dom_pk>"
}
```

Reglas:

- `exp - iat` DEBE ser ≤ 30 días (2 592 000 s).
- Una prueba caducada no vale nada: la entrada que la lleva no es válida.
- Una `attestation` solo es válida si `verifier_pk` está en la lista `verifiers` de la tabla de shards de la federación que usa el cliente (§6).
- Una `domainsig` solo es válida si `sig` verifica contra la `dom_pk` publicada en el DNS de `domain` (§5) y `domain` coincide con el dominio del email.

### 4.2 Entrada de dispositivo

La unidad del directorio. El propio dispositivo la construye, la firma y la publica.

```json
{
  "type": "device",
  "v": 1,
  "uid": "<uid>",
  "dev_pk": "<dev_pk>",
  "dev_dh": "<dev_dh>",
  "name": "Laptop",
  "homes": ["https://node.7kas.com"],
  "transports": ["noise", "webrtc"],
  "endpoints": ["udp://203.0.113.7:7700"],
  "apps": ["7ks-chat/1", "7ks-call/1", "7ks-file/1"],
  "seq": 3,
  "exp": 1765152000,
  "proof": { "type": "domainsig", "uid": "<uid>", "dev_pk": "<dev_pk>", "domain": "7kas.com", "iat": 1762560000, "exp": 1765152000, "sig": "<…>" },
  "sig": "<firma con dev_pk>"
}
```

| Campo | Obligatorio | Significado |
|---|---|---|
| `v` | sí | Versión del formato: 1 |
| `uid`, `dev_pk`, `dev_dh` | sí | Identificadores (§3) |
| `name` | no | Nombre legible para que el usuario reconozca sus dispositivos. ≤ 64 caracteres |
| `homes` | sí | URL base HTTPS de los nodos a través de los cuales se localiza este dispositivo, por orden de preferencia. De 1 a 4 |
| `transports` | sí | Perfiles de transporte que soporta, por orden de preferencia: `noise`, `webrtc`. Al menos uno |
| `endpoints` | no | Direcciones UDP en las que acepta un handshake `noise` directo sin atravesar NAT (§11.7). De 0 a 4 |
| `apps` | sí | Protocolos de aplicación que acepta, `nombre/versión`. Al menos uno |
| `seq` | sí | Estrictamente creciente por `dev_pk`. Se recomienda usar el tiempo Unix de la publicación |
| `exp` | sí | Caducidad. NO DEBE superar `proof.exp` |
| `proof` | sí | Prueba de titularidad (§4.1) |
| `sig` | sí | Firma con la clave privada de `dev_pk` |

Validación que DEBEN hacer todo nodo que reciba una entrada y todo cliente que la use:

```
validate_entry(e):
    1. e.v == 1; campos obligatorios presentes; tipos correctos.
    2. verify(e, e.dev_pk)                                      // firma del dispositivo
    3. e.proof.uid == e.uid  y  e.proof.dev_pk == e.dev_pk
    4. ahora < e.proof.exp  y  ahora < e.exp  y  e.exp <= e.proof.exp
    5. según e.proof.type:
         attestation: e.proof.verifier_pk ∈ verifiers de la federación; verify(e.proof, verifier_pk)
         domainsig:   e.proof.domain == dominio(uid)*; verify(e.proof, dom_pk del DNS)
    6. en un nodo: e.seq > seq almacenado para e.dev_pk (si lo hay)
    * un nodo conoce el dominio por la ruta de escritura; un cliente, por el email que resolvió.
```

### 4.3 Registro de identidad

El conjunto de entradas vigentes de un `uid`. No tiene firma propia: cada entrada se verifica por separado.

```json
{ "type": "record", "uid": "<uid>", "devices": [ { "type": "device", "…": "…" } ] }
```

Un nodo DEBE omitir las entradas caducadas al servirlo. Un cliente DEBE descartar las entradas que no superen la validación y usar el resto; si ninguna la supera, el registro se trata como inexistente.

### 4.4 Registro de nodo

```json
{
  "type": "node",
  "v": 1,
  "nid": "<nid>",
  "node_pk": "<clave Ed25519 del nodo>",
  "url": "https://node.7kas.com",
  "stun": "node.7kas.com:3478",
  "roles": { "domains": ["7kas.com"], "shards": [], "verifier": true, "relay": true },
  "seq": 12,
  "exp": 1765152000,
  "sig": "<firma con node_pk>"
}
```

| Campo | Significado |
|---|---|
| `url` | Base de toda la API del nodo (§9). Un nodo tiene una única URL base; la redundancia se consigue con DNS y balanceadores de carga, fuera del protocolo |
| `stun` | Servidor STUN del nodo, `host:puerto`. Todo nodo DEBE operar uno (§11.2) |
| `roles.domains` | Dominios de los que es autoritativo |
| `roles.shards` | Prefijos del directorio público que sirve (informativo; la autoridad es la tabla de shards) |
| `roles.verifier` | Si acepta peticiones de verificación (§8) |
| `roles.relay` | Si ofrece reenvío TURN (§11.2) |

### 4.5 Tabla de shards

```json
{
  "type": "shards",
  "v": 1,
  "fed_id": "7ks",
  "nodes": { "<nid>": "https://shard3.example", "<nid>": "https://shard9.example" },
  "prefixes": [ ["<nid>", "<nid>", "<nid>"], ["<nid>", "<nid>", "<nid>"], "… 65536 listas …" ],
  "verifiers": ["<verifier_pk>", "<verifier_pk>"],
  "seq": 40,
  "exp": 1765152000,
  "prev_sig": "<firma con la fed_pk anterior, solo tras una rotación>",
  "sig": "<firma con fed_pk>"
}
```

- `prefixes[i]` es la lista de `nid` responsables del prefijo `i`, de 0 a 65535. Cada lista DEBERÍA tener al menos 3 nodos de operadores distintos.
- `nodes` asocia cada `nid` a su URL base; su `node_pk` se obtiene con `GET /node` en esa URL (§9.1).
- `verifiers` son las claves cuyas atestaciones acepta la federación.
- `sig` es de `fed_pk`. `prev_sig`, presente durante 90 días tras una rotación, es la firma con la `fed_pk` anterior, para que un cliente que tiene la clave antigua pueda aceptar la tabla que anuncia la nueva.
- Tamaño orientativo: 65536 × 3 referencias ≈ 2 MB en JSON. Un cliente la descarga una vez y la refresca por `seq` (§9.1).

### 4.6 Vigilancia (watch)

Lo único que un nodo almacena de forma persistente sobre el tráfico. Vive en el nodo del destinatario.

```json
{ "watcher_uid": "<uid>", "watcher_domain": "example.com", "watcher_home": "https://nodeA.example", "target_uid": "<uid>", "exp": 1763164800 }
```

- `watcher_domain` es el dominio de la dirección del vigilante; el nodo del destinatario lo usa para comprobar que el nodo que registra la vigilancia es un home del vigilante (§9.4).
- `watcher_home` es la URL en la que el nodo del vigilante quiere recibir `online`: uno de los `homes` de la entrada del vigilante.

---

## 5. Dominios

### 5.1 Entrada DNS

Un dominio se une a la red publicando un registro TXT:

```
_7ks.<dominio>.   TXT   "v=7ksp1; url=https://node.<dominio>; pk=<dom_pk en base64url>"
```

Formato: pares `clave=valor` separados por `;` y espacios opcionales. `v` DEBE ser `7ksp1`; `url` es la base HTTPS del nodo de dominio; `pk` es `dom_pk`. Un registro al que le falte cualquiera de los tres, o con un `v` desconocido, se ignora como si no existiera.

Alternativa para quien no puede publicar TXT: `https://<dominio>/.well-known/7ks`, devolviendo

```json
{ "v": "7ksp1", "url": "https://node.example.com", "pk": "<dom_pk>" }
```

Si existen ambos, gana el TXT. Un cliente DEBERÍA cachear el resultado respetando el TTL del DNS y en ningún caso durante más de 24 horas; una respuesta negativa (sin entrada) se cachea como máximo 1 hora.

### 5.2 Autoridad

Un dominio con una entrada vigente es **autoritativo** sobre sus direcciones:

- Solo su nodo publica entradas de sus usuarios, siempre con `domainsig`.
- Un nodo de shard NO DEBE servir entradas de un dominio con una entrada DNS vigente.
- Un cliente NO DEBE aceptar, para ese dominio, entradas con `attestation`.
- Si el nodo del dominio responde `404` para un `uid`, ese usuario no existe. El cliente NO DEBE consultar el directorio público.

Un nodo PUEDE servir varios dominios, cada uno con su propia `dom_pk`. Un dominio PUEDE cambiar de nodo publicando una URL nueva; sus entradas siguen siendo válidas porque las firma `dom_pk`, no el nodo.

### 5.3 Entrada y salida

Un dominio cuyos usuarios estaban en el directorio público pasa a ser autoritativo con solo publicar el DNS. A partir de ese momento dejan de aceptarse las entradas públicas de ese dominio, que caducan solas. Los contactos que tenían claves fijadas reciben el aviso de cambio de `dom_pk` (de "ninguna" a una). Si el dominio retira el DNS, sus usuarios pueden volver a publicar en el directorio público con `attestation`. Entrar y salir es decisión del titular del dominio.

---

## 6. Federación

Una federación es: un `fed_id`, una `fed_pk`, una tabla de shards firmada y una URL de arranque desde la que descargarla.

El cliente se configura con una o varias federaciones (`fed_id`, `fed_pk`, URL de arranque). El protocolo no distingue entre ellas; el Anexo A describe la federación pública de 7ks, que es la predeterminada en el cliente 7ks. Cualquiera PUEDE operar una federación distinta con el mismo software.

Obligaciones de un nodo de shard:

- Mantener la tabla al día: descargarla de la URL de arranque o de cualquier otro nodo de shard, y aceptarla solo si `sig` verifica contra `fed_pk` (o `prev_sig` contra la anterior durante una rotación) y su `seq` es mayor que el de la que tiene.
- Replicar sus prefijos: como máximo cada 5 minutos, pedir a los demás nodos de cada uno de sus prefijos las entradas que han cambiado (`GET /sync`, §9.4) y aceptar las que superen la validación.
- Servir las lecturas públicas (§9.1) sin autenticación.
- Aceptar escrituras (§9.2) solo para los prefijos que le asigna la tabla.

Obligaciones de un operador de federación, fuera del protocolo pero necesarias: publicar los criterios de admisión de nodos de shard, mantener la cobertura de los 65536 prefijos con al menos 3 nodos cada uno, publicar el procedimiento de rotación de `fed_pk` y un compromiso de neutralidad (no filtrar, priorizar ni descartar entradas válidas).

---

## 7. Resolución

Entrada: una dirección de email. Salida: su registro de identidad verificado, o la certeza de que no existe.

```
resolve(email):
    uid = sha256(normalize(email)); dominio = parte tras la '@'
    1. Si hay un registro en caché con entradas vigentes y la caché no ha caducado → devolverlo.
    2. Buscar la entrada DNS del dominio (§5.1).
       2a. Existe (url, dom_pk):
           GET <url>/7ks/v1/record/<uid hex>
           200 → quedarse con las entradas cuya prueba es una domainsig que valida contra dom_pk.
                 Si queda alguna → pin(uid, entradas, dom_pk); devolver.
                 Si no queda ninguna → tratar como 404.
           404 → NO EXISTE. Fin. No se consulta el directorio público.
           error de red → reintentar con espera creciente; si persiste, devolver "no disponible" (no "no existe").
    3. Sin entrada DNS:
       prefijo = uid[0..2]; candidatos = tabla.prefixes[prefijo], en orden aleatorio
       para cada nid de candidatos:
           GET <tabla.nodes[nid]>/7ks/v1/record/<uid hex>
           200 → quedarse con las entradas cuya prueba es una attestation de una verifier_pk ∈ tabla.verifiers y que validan.
                 Si queda alguna → pin(uid, entradas, sin dom_pk); devolver.
           404 → siguiente candidato.
           307 → seguir Location una vez y continuar.
           error → siguiente candidato.
       Si todos respondieron 404 → NO EXISTE. Si ninguno respondió → "no disponible".
```

**Resolución por un nodo.** Cuando un nodo resuelve en nombre de un dispositivo (`auth` de un dispositivo cuyo directorio no sirve, `resolve`, `connect`, `watch`), recibe del dispositivo `uid` y `domain`: el dominio no puede recuperarse del `uid` y sin él el paso 2 es imposible. El nodo ejecuta el mismo algoritmo con el par (`uid`, `domain`) en lugar de la dirección, valida las entradas exactamente como lo haría un cliente y PUEDE cachear el registro validado hasta el `exp` más pequeño de sus entradas y como máximo 1 hora. Un nodo NO DEBE consultar ni reenviar a ninguna URL que el algoritmo no haya producido: la `url` de la entrada DNS del dominio, los nodos que la tabla (o la convención de nombres del Anexo A) asigna al prefijo y los `homes` de entradas validadas, cada uno comprobado como en el §9.4.

### 7.1 Fijación de claves (TOFU)

```
pin(uid, entradas, dom_pk):
    conocido = almacen_local[uid]
    si conocido no existe:
        guardar {devices: {dev_pk → dev_dh}, dom_pk}; avisar a la app: "nuevo contacto"
    si existe:
        si conocido.dom_pk != dom_pk → avisar a la app: "posible suplantación: ha cambiado la clave del dominio";
                                       NO usar las entradas nuevas hasta que el usuario confirme
        por cada entrada con un dev_pk desconocido → añadir; avisar a la app: "nuevo dispositivo de <uid>"
        por cada dev_pk conocido que ya no aparece → marcar como retirado (no borrar: puede volver con un seq mayor)
```

### 7.2 Caché y refresco

- Un cliente DEBERÍA cachear un registro hasta el `exp` más pequeño de sus entradas y como máximo 24 horas.
- Un cliente DEBERÍA suscribirse (§9.3 `subscribe`) a los `uid` con los que habla con frecuencia, para recibir `record_changed` y volver a resolver sin esperar a la caducidad. Un nodo emite `record_changed` para los `uid` cuyo directorio sirve; para los demás no está garantizado, así que un cliente DEBERÍA volver a resolver a un contacto alojado en otro nodo cuando su registro en caché tenga más de 1 hora al iniciar una sesión con él, y tras cualquier `unknown` o fallo de validación.
- Un cliente PUEDE delegar los pasos 2 y 3 en su home con `resolve` (§9.3), pasando `uid` y `domain`. El home ejecuta el mismo algoritmo. El cliente DEBE validar igualmente el resultado: el home es un transporte, no una autoridad.

---

## 8. Alta y vinculación de dispositivos

Un único procedimiento da de alta a un usuario nuevo, añade un dispositivo a un usuario existente y recupera la cuenta tras perder todos los dispositivos. La prueba es siempre la misma: abrir un enlace enviado a la dirección de email.

### 8.1 Flujo

```
Dispositivo                                     Verificador                        Buzón del usuario
    |                                                |                                   |
    | genera dev_pk, dev_dh                          |                                   |
    |-- POST /7ks/v1/verify {email, dev_pk, dev_dh} ->|                                   |
    |<- 202 {req_id} --------------------------------|                                   |
    |                                                |-- email con enlace(token) -------->|
    |-- GET /7ks/v1/verify/<req_id> (espera 30 s) -->|                                   |
    |<- 202 (todavía no) ----------------------------|                                   |
    |-- GET /7ks/v1/verify/<req_id> ---------------->|                                   |
    |                                                |<- el usuario abre el enlace -------|
    |<- 200 {proof} ---------------------------------|                                   |
    | construye y firma su entrada de dispositivo    |                                   |
    |-- PUT /7ks/v1/record/<uid>/<dev_pk> {entrada} -> nodo responsable (dominio o shards)
    |<- 204 -----------------------------------------|
```

1. El dispositivo genera `dev_pk` y `dev_dh`.
2. Elige un verificador: el nodo de dominio si el dominio tiene entrada DNS; si no, cualquier nodo con `roles.verifier` cuya clave esté en los `verifiers` de la federación.
3. `POST /verify` con el email en claro (la única operación del protocolo que lleva el email) y sus claves públicas. Respuesta `202 {req_id}`.
4. El verificador envía un email con dos enlaces que llevan el mismo `token`: `https://<verificador>/7ks/v1/verify/<token>/confirm` para abrir en cualquier navegador, y `7ks:verify?token=<token>` para abrir directamente en la aplicación.
5. El dispositivo espera con `GET /verify/<req_id>`. El servidor PUEDE retener la respuesta hasta 30 s (long-poll); responde `202` si todavía no, `200 {proof}` cuando se ha usado el token y `410` si caducó.
6. El dispositivo construye su entrada de dispositivo con la `proof`, la firma y la publica con `PUT /record` (§9.2) en el nodo responsable: el nodo de dominio, o uno de los nodos de shard del prefijo (que la replica a los demás).
7. El dispositivo abre una sesión en su home (§9.3). Ya está en línea.

Reglas sobre los identificadores:

- `req_id` y `token` DEBEN ser valores aleatorios distintos de al menos 128 bits. `req_id` lo conoce el dispositivo; `token`, solo quien lee el email. Así, quien intercepta la petición no puede confirmarla, y quien lee el email no puede recoger la prueba sin el dispositivo.
- `token` caduca a los 15 minutos. `req_id` deja de poder consultarse 5 minutos después de entregar la prueba.

### 8.2 Contenido del email

El verificador DEBERÍA enviar un email que contenga: el nombre del dispositivo si se indicó, los dos enlaces, el plazo de validez y una frase diciendo que, si el usuario no ha pedido nada, puede ignorarlo. NO DEBE incluir las claves ni nada que identifique la petición más allá del token.

### 8.3 Obligaciones del verificador

- Enviar el enlace solo a la dirección indicada, nunca a otra.
- Emitir la prueba solo después de que se haya abierto el token.
- Limitar la frecuencia: DEBERÍA rechazar más de 5 peticiones por dirección y hora (`429`), y aplicar límites por origen de red.
- No conservar el email en claro tras emitir la prueba; conservar solo el `uid`.
- Firmar `iat` con su propio reloj; `exp = iat + 30 días` como máximo.

### 8.4 Renovación

Las entradas caducan a los 30 días. A los 20 días el dispositivo DEBERÍA renovar: en un dominio, el nodo PUEDE renovar la `domainsig` sin un enlace nuevo (decisión del dominio); en el directorio público, se repite el flujo del §8.1 con la misma `dev_pk`. Un dispositivo que no renueva desaparece del directorio sin más consecuencias; cuando vuelve, repite el alta.

### 8.5 Desvinculación

```
DELETE /7ks/v1/record/<uid hex>/<dev_pk>
Cuerpo:
{ "type": "revoke", "uid": "<uid>", "dev_pk": "<dev_pk a eliminar>", "by": "<clave firmante>", "iat": 1762560000, "sig": "<…>" }
```

`sig` DEBE ser de una de estas claves: otro dispositivo vigente del mismo `uid` (`by` = su `dev_pk`), un verificador aceptado tras un nuevo enlace (`by` = `verifier_pk`) o el dominio (`by` = `dom_pk`). `iat` DEBE estar a menos de 5 minutos del reloj del nodo. El nodo DEBE borrar la entrada, propagarlo a sus réplicas y notificar `record_changed` (§9.3) a los clientes conectados suscritos a ese `uid`.

### 8.6 Recuperación

Quien ha perdido todos sus dispositivos repite el alta en uno nuevo: obtiene una prueba nueva y publica una entrada nueva. Las entradas de los dispositivos perdidos siguen siendo válidas hasta que caducan; el usuario DEBERÍA revocarlas desde el dispositivo nuevo (§8.5). Los contactos reciben "nuevo dispositivo" y, al revocarse, `record_changed`.

### 8.7 Dominios

Un nodo de dominio PUEDE sustituir el flujo del enlace por lo que decida el dominio (login corporativo, alta manual, gestión por un administrador), siempre que el resultado sea una `domainsig`. El resto del protocolo no cambia.

---

## 9. API del nodo

Toda la API vive bajo `<url>/7ks/v1/`. Las rutas de lectura son HTTPS sin autenticar; las de escritura llevan objetos firmados; la sesión del dispositivo es un WebSocket. Los nodos usan HTTPS porque son servidores ordinarios con nombres y certificados; los dispositivos nunca lo usan entre sí.

### 9.1 Lecturas públicas

| Ruta | Respuesta |
|---|---|
| `GET /record/<uid hex>` | `200` registro de identidad · `404` · `307 Location` si el nodo no es responsable y sabe quién lo es |
| `GET /node` | `200` registro de nodo |
| `GET /shards` | `200` tabla de shards (solo nodos de shard) |
| `GET /shards?since=<seq>` | `200` tabla · `304` si no ha cambiado |

Estas rutas NO DEBEN exigir autenticación ni limitar por origen más allá de lo razonable contra abusos. Cada respuesta es verificable por sus firmas; un cliente NO DEBE fiarse de una respuesta por el nodo del que viene. `Content-Type: application/json`. Las respuestas DEBERÍAN llevar un `Cache-Control` coherente con las caducidades del §7.2.

### 9.2 Escrituras

| Ruta | Cuerpo | Respuesta |
|---|---|---|
| `PUT /record/<uid hex>/<dev_pk>` | entrada de dispositivo (§4.2) | `204` · `400 invalid` · `409 seq` · `403 not_responsible` |
| `DELETE /record/<uid hex>/<dev_pk>` | revocación firmada (§8.5) | `204` · `400 invalid` · `403` |
| `POST /verify` | `{ "email": "…", "dev_pk": "…", "dev_dh": "…", "name": "…" }` | `202 {req_id}` · `429 limit` · `403` si el nodo no es verificador de ese dominio |
| `GET /verify/<req_id>` | — | `200 {proof}` · `202` · `410 expired` |
| `GET /verify/<token>/confirm` | — | página HTML de confirmación; marca el token como usado |

Un nodo es responsable de un `uid` si: es el nodo de dominio de ese dominio, o es un nodo de shard del prefijo y el dominio no tiene entrada DNS. En cualquier otro caso responde `403` con `{ "error": "not_responsible", "responsible": ["<url>", "…"] }` si lo sabe. `PUT` y `DELETE` no necesitan sesión: el objeto lleva su propia autoridad. Un nodo DEBE rechazar (`invalid`) una entrada cuyos `homes` contengan una URL que no sea `https` o cuyo host no supere la regla de direcciones del §9.4, y DEBERÍA además exigir un registro de nodo válido en cada una (§9.4), porque otros nodos reenviarán a esas URL.

### 9.3 Sesión del dispositivo (WebSocket)

Un dispositivo conectado a un nodo abre `wss://<host de url>/7ks/v1/session` con el subprotocolo `Sec-WebSocket-Protocol: 7ksp.v1`. Mientras la sesión está abierta, el dispositivo está **en línea** en ese nodo. Cerrarla es desconectarse. No hay otro mecanismo de presencia.

Todos los mensajes son objetos JSON con `t` (tipo). Las peticiones llevan `id` (una cadena elegida por el emisor, única dentro de la sesión) y la respuesta la repite. Los mensajes no solicitados del nodo no llevan `id`.

**Autenticación.** El nodo abre con un desafío; el dispositivo responde firmado.

```
← { "t": "challenge", "nonce": "<32 bytes aleatorios>" }

→ { "t": "auth",
    "uid": "<uid>", "domain": "7kas.com", "dev_pk": "<dev_pk>",
    "nonce": "<el mismo>", "host": "node.7kas.com",
    "push": { "gateway": "https://push.7ks.app", "token": "<opaco>" },
    "sig": "<firma Ed25519 con dev_pk de JCS({uid, dev_pk, nonce, host})>" }

← { "t": "ok", "time": 1762560000,
    "stun": ["node.7kas.com:3478", "stun2.example:3478"],
    "relay": [ { "addr": "node.7kas.com:3478", "username": "<…>", "credential": "<…>", "exp": 1762563600 } ] }
   o
← { "t": "error", "code": "unauthorized" }   y cierra
```

- El nodo DEBE comprobar que `dev_pk` tiene una entrada vigente para `uid`: en su propio directorio si es responsable; resolviendo (§7) si no. Un nodo PUEDE aceptar como home dispositivos de cualquier dominio; no necesita ser responsable de su directorio.
- `domain` es el dominio de la dirección del dispositivo. No va firmado: un nodo que sirve el directorio de ese dominio lo ignora; cualquier otro lo necesita para resolver (§7), y un valor erróneo hace fallar la resolución y la sesión es `unauthorized`.
- `host` impide que una firma sea válida en otro nodo. `nonce` impide la repetición.
- `push` es opcional (§13). Un dispositivo de escritorio no lo envía.
- `stun` enumera servidores STUN, al menos el del propio nodo. `relay` enumera servidores TURN que el dispositivo puede usar, con credenciales que DEBERÍAN caducar al cabo de 1 hora; el dispositivo PUEDE volver a autenticarse para obtener otras. Ambas listas PUEDEN incluir otros nodos que anuncien `roles.relay`. En el perfil `webrtc` se pasan a la pila WebRTC como sus servidores ICE.
- Tras `ok`, el nodo DEBE recorrer las vigilancias cuyo destinatario es este `uid` (§12) y actualizar la entrada `homes` si el dispositivo lo pide.

**Mensajes del dispositivo al nodo.**

| Mensaje | Respuesta | Función |
|---|---|---|
| `{ "t": "resolve", "id": "…", "uid": "…", "domain": "…" }` | `{ "t": "record", "id", "record" }` · `{ "t": "error", "id", "code": "not_found" }` · `{ "t": "error", "id", "code": "unavailable" }` | El nodo ejecuta §7 en nombre del dispositivo con (`uid`, `domain`) |
| `{ "t": "connect", "id": "…", "env": {…} }` | `{ "t": "sent", "id", "devices": ["<dev_pk>", …] }` · `{ "t": "offline", "id" }` · `{ "t": "error", "id", "code": "unknown" }` · `{ "t": "error", "id", "code": "unavailable" }` | Entregar un sobre de señalización (§10) a los dispositivos en línea del destinatario, a través de su home cuando es otro nodo (§9.4) |
| `{ "t": "watch", "id": "…", "uid": "…", "domain": "…", "ttl": 604800 }` | `{ "t": "ok", "id" }` · `{ "t": "error", "id", "code": "limit" }` · `{ "t": "error", "id", "code": "unavailable" }` | Avisar cuando `uid` esté en línea (§12) |
| `{ "t": "unwatch", "id": "…", "uid": "…" }` | `{ "t": "ok", "id" }` | |
| `{ "t": "subscribe", "id": "…", "uids": ["…"] }` | `{ "t": "ok", "id" }` | Recibir `record_changed` de esos `uid` durante la sesión |
| `{ "t": "block", "id": "…", "uid": "…" }` / `unblock` | `{ "t": "ok", "id" }` | El nodo descarta todo lo que llegue de `uid` a este usuario |
| `{ "t": "ping" }` | `{ "t": "pong" }` | Mantener viva la sesión; cada 30 s |

**Mensajes del nodo al dispositivo.**

| Mensaje | Función |
|---|---|
| `{ "t": "deliver", "env": {…} }` | Sobre de señalización recibido |
| `{ "t": "online", "uid": "…" }` | Un `uid` al que este dispositivo esperaba está en línea |
| `{ "t": "record_changed", "uid": "…" }` | El registro de un `uid` suscrito ha cambiado; volver a resolver |

Semántica de `connect`:

- `sent` indica a qué dispositivos se entregó el sobre (los que tienen sesión) o a cuáles se envió un push con el sobre retenido (§13). No garantiza que lo hayan leído.
- `offline`: ningún dispositivo del destinatario tiene sesión ni push registrado. El sobre se descarta.
- `unknown`: el `uid` no existe o ha bloqueado al origen. Son indistinguibles por diseño.
- `unavailable`: no se pudo contactar con el nodo del destinatario o no respondió a tiempo (§9.4). No se entregó nada; el cliente guarda el contenido y reintenta más tarde. No es `offline`: una `watch` tampoco llegaría a ese nodo.
- Un nodo DEBE validar que `env.from.uid` y `env.from.dev` coinciden con la sesión que envía y que `env.from.domain` es el dominio con el que esa sesión se autenticó; si no, `error invalid`.

### 9.4 Entre nodos

Los nodos se comunican entre sí por HTTPS con los mismos objetos. Un nodo llama a otro solo en cuatro situaciones: es el home de un dispositivo que envía a un usuario del otro nodo (`POST /relay`); es el home de un vigilante (`POST /watch`); es el home de un destinatario al que alguien vigila (`POST /online`); o ambos son nodos de shard del mismo prefijo (`GET /sync`). Cada petición entre nodos lleva:

```
Authorization: 7ks-node <nid> <url> <ts> <sig>
sig = base64url( Ed25519_sign( node_pk, JCS({ "method": "POST", "path": "/7ks/v1/relay", "host": "bfc9.7ks.app", "ts": 1762560000, "body_sha256": "<base64url>" }) ) )
```

- `url` es la URL base del llamante como nodo: la `url` de su registro de nodo (§4.4) o un alias de ella según la convención de nombres de su federación (Anexo A). El llamante usa el nombre suyo que aparece en los `homes` del usuario por el que habla.
- `host` es el nombre del receptor tal como lo direccionó el llamante (el host de la URL que usó). El receptor DEBE comprobar que es uno de sus nombres; una petición firmada para otro nodo es `401`.
- `ts` es el reloj del llamante, en segundos. El receptor DEBE rechazar una diferencia de más de 5 minutos (`401`). Los nodos mantienen el reloj sincronizado (NTP).
- El receptor DEBERÍA recordar (`nid`, `ts`, `body_sha256`) durante 5 minutos y rechazar una repetición (`401`).

**Asociar `nid` a un nodo.** Antes de confiar en una firma, el receptor DEBE obtener `GET <url>/7ks/v1/node`, comprobar que el registro valida, que `nid = sha256(node_pk)` y que la `url` del registro es `url` o que `url` es un alias suyo según la convención de nombres. PUEDE cachear la asociación hasta el `exp` del registro y como máximo 24 horas; cuando una firma falla contra una asociación cacheada, vuelve a obtener el registro una vez antes de rechazar. La asociación demuestra quién llama, no que pueda hacerlo: lo decide el contenido.

| Ruta | El llamante es legítimo cuando `url` … |
|---|---|
| `POST /relay` | es uno de los `homes` de la entrada `env.from.dev` en el registro validado de (`env.from.uid`, `env.from.domain`), resuelto según el §7 |
| `POST /watch` | es igual a `watcher_home` y es uno de los `homes` del vigilante en el registro validado de (`watcher_uid`, `watcher_domain`) |
| `POST /online` | es el nodo al que este receptor envió la vigilancia correspondiente: un nodo conserva las vigilancias que ha reenviado (vigilante, destinatario, nodo del destinatario) hasta que caducan y solo acepta `online` para una de ellas |
| `GET /sync` | es un nodo que la tabla de shards asigna a ese prefijo |

Una petición cuyo llamante no es legítimo es `403`. Esto es lo que da sentido a los límites por nodo llamante: un nodo solo puede hablar por los usuarios de los que es home, y un desconocido que opere un nodo no puede inundar a otro en nombre de usuarios inventados.

| Ruta | Cuerpo | Función |
|---|---|---|
| `POST /relay` | `{ "env": {…} }` | Entregar un sobre a un dispositivo de este nodo. Responde como `connect`: `{ "status": "sent" \| "offline" \| "unknown", "devices": [...] }` |
| `POST /watch` | `{ "watch": {…} }` | Registrar una vigilancia (§4.6) en este nodo, que es el nodo del destinatario |
| `POST /online` | `{ "uid": "<destinatario>", "watcher_uid": "<uid>" }` | Avisar a este nodo de que un destinatario al que esperaba uno de sus usuarios está en línea |
| `GET /sync/<prefijo>?since=<ts>` | — | Entradas y revocaciones del prefijo modificadas desde `since` (solo entre nodos de shard del mismo prefijo) |

**Reenvío.** Un nodo que recibe un `connect` o una `watch` para un `uid` del que no es home DEBE resolver (§7) con el `domain` que le dio el dispositivo, tomar los `homes` de la entrada destinataria (`to.dev`), o de todas las entradas cuando `to.dev = "*"`, y reenviar exactamente una vez al primer home que responda, en el orden de la entrada. Un `POST /relay` recibido nunca se reenvía. Antes de contactar con un home el nodo DEBE comprobar que es un nodo: esquema `https`; un nombre de host que resuelve a una dirección unicast pública, nunca de loopback, privada, de enlace local, de NAT de operador, multicast ni reservada (la regla de direcciones); y un registro de nodo válido en `<home>/7ks/v1/node` cuya `url` sea ese home o de la que ese home sea alias, cacheado como arriba. Un home que falla cualquier comprobación se salta. El nodo no inicia un `/relay` cuando faltan menos de 5 s para `env.exp`, y responde `unavailable` cuando ningún home respondió. La respuesta del home (`sent`, `offline`, `unknown`) se pasa al dispositivo sin cambios.

**Tiempos y límites.** Conexión 5 s, petición completa 10 s; como máximo 4 homes por entrada, un intento por home y sobre. Un nodo DEBERÍA mantener abiertas las conexiones con los nodos frecuentes, DEBERÍA dejar de llamar durante 30 s a un nodo que ha fallado repetidamente y responder `unavailable` mientras tanto, y DEBERÍA limitar las peticiones por nodo llamante y por `from.uid`.

**Qué necesita un nodo para hablar con otros nodos.** Su clave de nodo y su registro de nodo firmado servido en `/node` (§4.4), con `exp` renovado antes de que venza; los parámetros de su federación (§6) y la tabla vigente o la convención de nombres; un resolutor DNS con las cachés del §5.1; un cliente HTTPS que verifica certificados, sigue como máximo un `307` y aplica la regla de direcciones; las cachés del §7 (registros validados, como máximo 1 hora) y de esta sección (registros de nodo y asociaciones, como máximo 24 horas); la memoria de repeticiones; las vigilancias que almacena (§4.6) y las que ha reenviado.

---

## 10. Señalización

La señalización es lo único que pasa por los nodos. Lleva lo que los dos dispositivos necesitan para encontrarse y para vincular el canal a sus claves; nada más.

### 10.1 Sobre

Lo que transporta el nodo. El nodo ve el destinatario, el origen y la caducidad; el contenido es opaco.

```json
{
  "id": "<uuid v4>",
  "to":   { "uid": "<uid>", "domain": "example.com", "dev": "<dev_pk> | *" },
  "from": { "uid": "<uid>", "domain": "7kas.com", "dev": "<dev_pk>" },
  "exp": 1762560120,
  "body": "<base64url>"
}
```

- `to.dev = "*"`: el nodo entrega a todos los dispositivos en línea del `uid`. Atención: el cuerpo está sellado para un `dev_dh` concreto, así que un sobre con `*` solo puede abrirlo el dispositivo para el que se selló; `*` sirve para no tener que saber de antemano cuáles están en línea, no para enviar un mismo cuerpo a varios.
- `exp`: NO DEBE superar `ahora + 120 s`. Un nodo que no puede entregar antes de `exp` descarta el sobre. **Un nodo NO DEBE encolar sobres**: si el destinatario no está en línea en ese momento, responde `offline`. La única retención permitida es la del push (§13).
- `body` ≤ 16 KB.
- `id` permite a un receptor descartar duplicados durante 5 minutos.
- `to.domain` y `from.domain` son los dominios de las dos direcciones, en minúsculas. Un `uid` no puede invertirse y la resolución (§7) necesita el dominio: el home del emisor usa `to.domain` para encontrar el nodo del destinatario; el receptor usa `from.domain` para verificar `from_dev` (§10.2); el nodo del destinatario lo usa para comprobar quién llama (§9.4). El nodo emisor comprueba `from.domain` contra la sesión. La dirección en sí viaja solo dentro del cuerpo sellado, en el protocolo de aplicación, y el receptor comprueba que su hash es `from.uid` antes de mostrarla.

### 10.2 Cuerpo

```
inner = JCS({ "signal": <Señal>, "from_dev": "<dev_pk del emisor>", "sig": "<firma Ed25519 del emisor sobre JCS(signal)>" })
body  = sealed_box( dev_dh del destinatario, inner )
```

El receptor DEBE, en este orden:

1. Abrir la sealed box con su clave privada X25519. Si falla, descartar.
2. Comprobar que `from_dev` es una `dev_pk` vigente de `from.uid`; si no la tiene en caché, resolver (§7) con `from.domain`. Si no lo es, descartar.
3. Verificar `sig` con `from_dev`. Si falla, descartar.

Un cuerpo que no supera la validación nunca se responde. Como el cuerpo va firmado por `dev_pk` y solo lo puede leer el destinatario, todo lo que contiene queda vinculado a la identidad del emisor. Esto es lo que permite a los dos perfiles autenticar el canal sin certificados: el perfil `noise` apoya en ello la clave estática del handshake; el perfil `webrtc`, la huella DTLS.

### 10.3 Señales

```json
{ "type": "offer", "sid": "<16 bytes aleatorios, base64url>",
  "apps": ["7ks-call/1", "sip/2"],
  "transports": ["noise", "webrtc"],
  "noise":  { "ice": { "ufrag": "<4–256 caracteres>", "pwd": "<22–256 caracteres>", "candidates": ["<cand>", "…"],
                       "prange": { "start": 61236, "count": 16 } } },
  "webrtc": { "sdp": "<oferta SDP completa>" } }

{ "type": "answer", "sid": "<el mismo>", "app": "7ks-call/1", "transport": "noise",
  "noise":  { "ice": { "ufrag": "…", "pwd": "…", "candidates": ["…"], "prange": { "…": "…" } } } }

{ "type": "reject", "sid": "…", "reason": "declined | busy | no_app | no_transport | blocked" }

{ "type": "ice", "sid": "…", "candidate": "<cand>", "mid": "<solo webrtc>" }

{ "type": "bye", "sid": "…", "reason": "cancel | answered_elsewhere | failed | hangup" }
```

| Campo | Reglas |
|---|---|
| `sid` | Lo genera el iniciador, aleatorio, 128 bits. Todas las señales de una sesión llevan el mismo `sid`; las que tienen un `sid` desconocido o terminado se descartan |
| `apps` | Protocolos de aplicación que propone el iniciador, por orden de preferencia; DEBEN estar en las `apps` de la entrada del destinatario, si no la oferta se responde con `reject no_app`. El receptor elige uno. La aplicación del receptor decide si suena, acepta en silencio o rechaza, según la `app`; 7KSP solo entrega |
| `transports` | Perfiles que ofrece el iniciador para esta sesión, por orden de preferencia; DEBEN ser un subconjunto de la intersección de los `transports` de ambas entradas. Por cada perfil ofrecido, la oferta lleva sus parámetros bajo su nombre. La respuesta nombra el `transport` elegido y lleva solo sus parámetros. Sin perfil común → `reject no_transport` |
| `noise.ice.candidates` | Cadenas de candidato ICE con la sintaxis `candidate:` de la RFC 8839, sin el prefijo `a=candidate:`. Ejemplo: `1 1 udp 2130706431 192.0.2.4 50000 typ host` |
| `noise.ice.prange` | Opcional. Rango de puertos públicos que el emisor prevé para sí mismo cuando está detrás de un NAT simétrico (§11.2). `count` ≤ 20 |
| `webrtc.sdp` | SDP generado por la pila WebRTC. Su línea `a=fingerprint` es la huella DTLS que el §11.5 vincula a la identidad |
| `ice` | Candidatos descubiertos después de enviar la oferta o la respuesta (trickle). `mid` solo en el perfil `webrtc` |
| `bye.reason` | `cancel` antes de una respuesta; `answered_elsewhere` a los demás dispositivos cuando uno responde; `failed` si no se pudo establecer el canal; `hangup` al terminar |

### 10.4 Varios dispositivos

Una `offer` dirigida a un usuario con varios dispositivos se envía **una vez por dispositivo**, cada sobre sellado para el `dev_dh` correspondiente y con el mismo `sid`. Cuando llega la primera `answer`, el iniciador DEBE enviar `bye answered_elsewhere` a los demás y descartar sus respuestas posteriores. Un receptor que recibe `bye` deja de sonar.

### 10.5 Máquina de estados de la sesión

```
Iniciador
  IDLE ──oferta enviada──► OFFERED ──answer──► CONNECTING ──canal listo──► CONNECTED ──bye/caída──► CLOSED
                             │ reject / 60 s sin respuesta → CLOSED (bye cancel)
                                                  │ 30 s sin canal → CLOSED (bye failed)
Receptor
  IDLE ──oferta recibida──► RINGING ──el usuario acepta (answer)──► CONNECTING ──canal listo──► CONNECTED ──► CLOSED
                              │ reject / bye / 60 s → IDLE
```

En CONNECTED, la señalización a través del nodo solo se usa si se pierde el canal directo y hay que renegociarlo (§11.8); el `bye` de cierre PUEDE ir por el canal directo o a través del nodo.

### 10.6 Temporizadores

| Evento | Valor |
|---|---|
| Sin `answer` tras `offer` | 60 s → `bye cancel` |
| Sin canal directo tras `answer` | 30 s → `bye failed` |
| Reenvío de candidatos `ice` | según los descubre el agente ICE (trickle); sin reenvío periódico |
| Memoria de `id` de sobres para duplicados | 5 minutos |

---

## 11. Canal directo

### 11.1 Qué garantiza

Sea cual sea el perfil, una vez establecido el canal le da a la aplicación:

- Confidencialidad e integridad de cada paquete.
- Autenticación mutua a nivel de dispositivo: los dos extremos son exactamente los dispositivos cuyas claves publicó el directorio. Nadie sin la clave privada de un dispositivo puede ocupar su lugar.
- Secreto hacia adelante: comprometer después la clave de larga duración de un dispositivo no expone las sesiones pasadas.
- Ningún nodo en el camino. Un relay, cuando se usa, reenvía paquetes que no puede leer.
- Dos primitivas: datagramas no fiables y streams fiables y ordenados (§11.6). Pistas de medios en el perfil `webrtc`.

### 11.2 Atravesar el NAT

Los dos dispositivos están detrás de NAT. Cada uno crea una asociación saliente en su propio NAT y después ambos usan esas asociaciones el uno hacia el otro. El procedimiento es simétrico: los dos lados hacen lo mismo a la vez. Es ICE (RFC 8445) restringido a UDP. En el perfil `noise` el cliente ejecuta ICE por sí mismo; en el perfil `webrtc` lo ejecuta la pila WebRTC, configurada con los mismos servidores STUN y relay.

Cómo funciona, en cuatro pasos:

```
1. A envía un paquete a un servidor STUN desde su socket UDP local (por ejemplo, el puerto 50000).
   Su NAT crea la asociación 50000 → 203.0.113.5:61234 y el servidor le dice a A "te veo en 203.0.113.5:61234".
   B hace lo mismo y obtiene 198.51.100.9:40011.
2. A y B se intercambian esas direcciones mediante la señalización (candidatos).
3. A, desde el MISMO socket, envía sondas a 198.51.100.9:40011. B, desde el MISMO socket, envía sondas a 203.0.113.5:61234.
   Los dos a la vez, cada 50–200 ms. El primer paquete saliente de cada uno autoriza en su NAT el entrante del otro.
4. Cuando una sonda obtiene respuesta en los dos sentidos, el par queda validado. El servidor ya no interviene.
```

Un cliente (o su pila WebRTC) DEBE:

- Ejecutar **ICE completo**, no ICE-lite. Reunir candidatos host (todas las direcciones locales, IPv6 incluida), candidatos server-reflexive (su dirección pública según cada servidor STUN de `stun`) y candidatos relay (asignaciones en servidores de `relay`), en ese orden de preferencia.
- Usar **un único socket UDP** para todo: consultas STUN, comprobaciones de conectividad y el propio canal. La asociación pública solo es válida para ese socket.
- Enviar comprobaciones de conectividad (peticiones STUN Binding con `ufrag`/`pwd`, integridad del mensaje incluida) a cada candidato del otro extremo, desde su propio socket, cada 50–200 ms, durante 30 s como máximo. **Los dos lados sondean a la vez.**
- Aceptar **candidatos peer-reflexive**: una sonda válida que llega desde una dirección no anunciada se convierte en un candidato nuevo.
- En el perfil `noise`, detectar un **NAT simétrico** en su lado comparando las asociaciones que devuelven dos servidores STUN o dos sockets. Si los puertos difieren, anunciar `prange` con los siguientes puertos previstos (normalmente consecutivos) y `count` ≤ 20. Un dispositivo que recibe `prange` DEBE enviar sondas también a esos puertos.
- Mantener vivo el par elegido cada 15–25 s (un paquete keepalive vacío en `noise`; en `webrtc` lo hace la propia pila WebRTC).
- No usar nunca TCP para el canal directo. Si la red bloquea UDP, el único camino es un relay alcanzado por TCP/TLS en el puerto 443, que sigue transportando paquetes UDP.

El iniciador es el agente ICE *controlling*: si se validan varios pares, elige cuál se usa y el receptor acepta la elección.

Un nodo DEBE operar un servidor STUN (RFC 8489) y anunciarlo en su registro. Un nodo DEBERÍA operar un servidor TURN (RFC 8656) con cuotas por sesión y anunciar `roles.relay`; DEBERÍA aceptar también TURN sobre TCP y TLS en el puerto 443 para las redes que bloquean UDP. Un relay solo ve paquetes cifrados y no puede leerlos ni alterarlos. Sirve cualquier software TURN estándar.

Resultado esperado: en la gran mayoría de las sesiones el canal es directo; el relay hace falta cuando los dos lados están detrás de NAT simétrico y la predicción de puertos falla.

### 11.3 Perfiles de transporte y negociación

| Perfil | Pila | Vinculación a la identidad | Quién lo implementa |
|---|---|---|---|
| `noise` | ICE → UDP → `Noise_IK` → datagramas y streams (§11.4, Anexo C) | la clave estática del handshake es la `dev_dh` del directorio | los clientes nativos DEBEN |
| `webrtc` | ICE → UDP → DTLS → canales de datos y pistas de medios (§11.5) | la huella DTLS llega dentro de la señal firmada y sellada | los clientes web DEBEN; los nativos DEBERÍAN |

Negociación: cada dispositivo anuncia sus perfiles en el campo `transports` de su entrada. El iniciador ofrece la intersección en su orden de preferencia; el receptor elige uno en la `answer`. Dos dispositivos nativos normalmente acaban en `noise`; una sesión con un navegador en cualquiera de los dos lados usa `webrtc`. Un cliente nativo que solo implementa `noise` es un cliente 7KSP completo; simplemente no puede hablar con navegadores.

La aplicación no ve el perfil. Abre streams y envía datagramas; el perfil los materializa (§11.6).

### 11.4 Perfil `noise`

#### Handshake

Sobre el par UDP elegido por ICE, los dos dispositivos ejecutan `Noise_IK_25519_ChaChaPoly_BLAKE2s` (Noise Protocol Framework, revisión 34) con el prólogo `"7ksp/1"` (bytes ASCII, sin terminador). Las claves estáticas son las `dev_dh` de los dispositivos. El iniciador ya conoce la `dev_dh` del receptor por el directorio; el receptor conoce la clave estática del iniciador al abrir el primer mensaje y la comprueba contra el directorio.

```
→ e, es, s, ss     handshake_init   iniciador: clave efímera; cifra su clave estática y una marca de tiempo
← e, ee, se        handshake_resp   receptor: clave efímera; los dos derivan las claves de sesión
```

Reglas:

- El iniciador del handshake es el iniciador de la sesión (`offer`).
- El receptor DEBE verificar que la clave estática descifrada es una `dev_dh` del `from.uid` de la oferta (el de la señalización sellada, que confirma el directorio). Si no lo es, DEBE descartar el mensaje sin responder.
- El iniciador DEBE verificar que el mensaje del receptor se descifra con las claves derivadas de la `dev_dh` que esperaba. Un receptor equivocado no puede producirlo.
- La carga útil de `handshake_init` es una marca de tiempo TAI64N de 12 bytes. El receptor DEBE rechazar una marca anterior a la última aceptada de esa clave estática (protección contra repetición) y DEBERÍA rechazar una que se aleje más de 90 s de su reloj.
- Bajo carga, un receptor PUEDE ignorar los `handshake_init` que superen un límite por dirección de origen; el iniciador retransmite cada 1 s hasta 5 veces.
- Renovación de claves: un handshake nuevo sobre la misma sesión cada 120 s, o tras 2⁶⁰ paquetes, o cuando cualquiera de los dos lados quiera. Las claves antiguas siguen siendo válidas durante 10 s después de establecerse las nuevas.

En el handshake no intervienen certificados, nombres ni firmas. La autenticación es Diffie-Hellman contra claves en las que los dos lados ya confían.

#### Formato de los paquetes

Todo paquete empieza por un byte de tipo. Los valores están elegidos para que un mismo socket pueda transportar a la vez STUN (primer byte 0–3), datos de canal TURN (64–79) y 7KSP (193–196), siguiendo la regla de demultiplexación de la RFC 7983.

```
0xC1  handshake_init   type(1) reserved(3) sender_idx(4) e(32) enc_s(48) enc_ts(28)          = 116 bytes
0xC2  handshake_resp   type(1) reserved(3) sender_idx(4) receiver_idx(4) e(32) enc_empty(16)  = 60 bytes
0xC3  data             type(1) reserved(3) receiver_idx(4) counter(8) AEAD(payload)           = 16 + payload + 16
0xC4  keepalive        = data con payload vacío                                               = 32 bytes
```

- `reserved`: tres bytes a cero. Un receptor DEBE ignorar su valor.
- `sender_idx` / `receiver_idx`: 4 bytes aleatorios por sesión que elige cada lado. Un receptor encuentra la sesión por `receiver_idx` sin descifrar; en un mismo socket conviven varias sesiones.
- `e`: clave pública X25519 efímera. `enc_s`: clave estática cifrada (32 + etiqueta de 16 bytes). `enc_ts`: marca de tiempo cifrada (12 + 16). `enc_empty`: carga útil vacía cifrada (solo la etiqueta).
- `counter`: 64 bits, estrictamente creciente por clave de sesión, que se usa como nonce del AEAD (12 bytes: 4 bytes a cero + los 8 del contador). El receptor DEBE descartar contadores repetidos o fuera de la ventana (ventana de 2048 hacia atrás).
- `AEAD(payload)`: ChaCha20-Poly1305 con la clave de sesión y el nonce; la etiqueta de 16 bytes va después del texto cifrado.
- Carga útil máxima: 1200 bytes, para evitar la fragmentación IP en cualquier camino (paquete máximo de 1232 bytes, por debajo de la MTU mínima de IPv6). Los datos de aplicación mayores los trocea la capa de streams.

#### Primitivas en la red

El primer byte de una carga útil descifrada selecciona la primitiva:

```
0x00  datagram   resto = bytes de la aplicación, entregados como una unidad, sin fiabilidad ni orden
0x01  stream     resto = una trama de stream (Anexo C), fiable y ordenada por stream
```

### 11.5 Perfil `webrtc`

El canal es una `PeerConnection` WebRTC estándar, tal como la implementan los navegadores y las librerías nativas (libdatachannel en C++, pion en Go, webrtc-rs en Rust, react-native-webrtc y flutter-webrtc en móvil).

- El iniciador crea la `PeerConnection` con los servidores `stun` y `relay` de su sesión como servidores ICE, genera una oferta SDP y la envía en `offer.webrtc.sdp`. El receptor la aplica, genera la respuesta y la envía en `answer.webrtc.sdp`. Los candidatos posteriores viajan en señales `ice` con `mid`.
- **Vinculación a la identidad.** La línea `a=fingerprint` del SDP llegó dentro de un cuerpo firmado por la `dev_pk` del emisor y legible solo por el destinatario. Al completarse DTLS, cada lado DEBE comparar la huella del certificado remoto con la del SDP que recibió y DEBE abortar si difieren. El certificado es autofirmado y no se valida contra nada más; no interviene ninguna autoridad de certificación. La garantía es la misma que en `noise`: sin la clave privada del dispositivo no se puede falsificar la señalización, y sin la clave privada del certificado no se puede completar el handshake DTLS.
- Una pila WebRTC genera su propio certificado autofirmado; el protocolo no le impone nada más allá de la comprobación de la huella. Un cliente DEBERÍA generar un certificado nuevo por sesión.
- El audio y el vídeo usan pistas WebRTC sobre SRTP, con claves derivadas del mismo handshake DTLS.

### 11.6 Primitivas

La aplicación pide lo mismo en los dos perfiles:

| Primitiva | `noise` | `webrtc` |
|---|---|---|
| Stream (fiable, ordenado, bidireccional) | stream del Anexo C | canal de datos con `ordered: true`, sin límite de retransmisiones |
| Datagrama (no fiable, sin orden) | carga útil datagram | canal de datos con `ordered: false, maxRetransmits: 0` |
| Pista de medios | no disponible; la aplicación envía los medios como datagramas | pista WebRTC |

La aplicación elegida en la `answer` es la dueña de la sesión. Por convención abre el primer stream como su stream de control y, en `webrtc`, etiqueta los canales de datos con el nombre de la aplicación (`createDataChannel("control", { protocol: "7ks-chat/1" })`). El formato que use dentro de los streams y los datagramas es cosa suya. 7KSP termina aquí.

### 11.7 Dispositivos alcanzables directamente

Un dispositivo con una dirección UDP pública (un servidor, un escritorio con un puerto redirigido) PUEDE anunciarla en `endpoints`. Un iniciador que soporta `noise` DEBE probar primero los `endpoints` anunciados, enviando `handshake_init` directamente, antes de iniciar ICE; si no llega ningún `handshake_resp` en 3 s, continúa con ICE como siempre. El handshake es el mismo, así que las garantías son las mismas. `endpoints` se aplica solo al perfil `noise`.

### 11.8 Reconexión

Si el par deja de funcionar (sin paquetes durante 3 intervalos de keepalive, o un cambio de red), cualquiera de los dos lados PUEDE reiniciar ICE con candidatos nuevos bajo el mismo `sid` durante 60 s como máximo. En `noise` la sesión se reanuda con un handshake nuevo; también PUEDE migrar a un par de direcciones nuevo sin handshake si llega desde él un paquete `data` válido (el contador demuestra que es el mismo extremo). En `webrtc` es un reinicio ICE de la misma `PeerConnection`. Pasados 60 s, un `sid` nuevo.

### 11.9 Navegadores

Un navegador no tiene sockets UDP y no puede ejecutar el perfil `noise`. Un cliente web implementa solo el perfil `webrtc` y anuncia `"transports": ["webrtc"]`. Es un cliente 7KSP completo: misma identidad, mismo directorio, misma señalización, mismas garantías.

### 11.10 Dispositivos móviles

Los sistemas operativos móviles cierran los sockets de una aplicación en segundo plano (iOS en segundos; Android con el ahorro de batería). Esto condiciona al cliente, no al protocolo:

- El canal directo vive mientras hay algo activo (una llamada, una transferencia). Cuando no lo hay, el cliente DEBERÍA cerrarlo y dejar de enviar keepalives.
- La presencia es la sesión WebSocket con el home (§9.3), no el canal entre dispositivos. Si el sistema no permite mantenerla, el dispositivo sigue siendo alcanzable por push (§13).
- Cuando lo despierta un push, el cliente abre una sesión, recibe el sobre retenido y reconstruye el canal desde cero. Con ICE y el handshake Noise esto lleva unos cientos de milisegundos.
- Un cliente DEBERÍA registrar un `push` en `auth`; un dispositivo de escritorio no lo necesita.

---

## 12. Entrega diferida

Sustituye a cualquier cola de mensajes. El nodo no almacena contenido; almacena quién espera a quién.

```
A1 (uid A)              Home de A (N)              Home de B (M)              B1 (uid B)
  |-- connect {→ B} ------>|                             |                          |
  |                        |-- POST /relay ------------->|  B no tiene sesión       |
  |                        |<-- {status: offline} -------|                          |
  |<-- offline ------------|                             |                          |
  | guarda el contenido en su almacén local              |                          |
  |-- watch {B, ttl} ----->|-- POST /watch {A, N, B} --->| guarda Watch(A→B)        |
  |<-- ok -----------------|                             |                          |
  …                                                                                 |
  |                        |                             |<-- auth (sesión) --------|
  |                        |<-- POST /online {B, A} -----| borra Watch(A→B)         |
  |<-- online {B} ---------|  (o push "online" si A no tiene sesión)                |
  |-- connect {→ B} ------>|-- POST /relay ------------->|-- deliver -------------->|
  …  canal directo A1 ↔ B1; A1 entrega lo que guardó
```

Reglas:

1. Cuando `connect` responde `offline`, el cliente guarda el contenido en su propio almacén y envía `watch` con `domain` y `ttl` ≤ 7 días. Su home lo reenvía al home del destinatario (resolviendo) como `POST /watch`, con `watcher_domain` y con `watcher_home` igual al nombre suyo que aparece en los `homes` del vigilante, y conserva una copia de lo reenviado para aceptar el `online` (§9.4).
2. Cuando un dispositivo del destinatario abre una sesión, su home recorre las vigilancias con ese `target_uid` y, para cada una: si el vigilante está conectado a este mismo nodo, le envía `online`; si no, `POST /online` a `watcher_home`, que lo entrega por la sesión o lo convierte en un push silencioso. La vigilancia se borra al emitirse.
3. El vigilante, al recibir `online`, hace `connect` y entrega por el canal directo.
4. Si el vigilante no recibió el aviso (estaba apagado y el push no lo despertó), reintenta al arrancar.
5. Un cliente DEBE reintentar todo lo pendiente al arrancar y al recuperar la conectividad, y DEBERÍA renovar las vigilancias pendientes antes de que caduquen.
6. Un nodo DEBE aceptar como máximo 256 vigilancias activas por `watcher_uid`; una repetida (mismo par vigilante/destinatario) renueva la existente en lugar de añadir otra.
7. `online` no revela nada que el vigilante no descubriría con un `connect`; aun así, un usuario PUEDE bloquear a otro y el nodo descartará sus vigilancias y sus sobres.

---

## 13. Push

Un dispositivo móvil dormido no tiene sesión abierta. Para alcanzarlo, el dispositivo registra en `auth` un `push` con la URL de una pasarela y un token opaco. La pasarela la elige el dispositivo, no el nodo. El nodo guarda `{dev_pk → pasarela, token}` mientras la entrada del dispositivo esté vigente.

Cuando un nodo necesita despertar a un dispositivo registrado sin sesión:

```
POST <pasarela>/7ks/v1/push
Content-Type: application/json
{ "token": "<opaco>", "kind": "connect" | "online" }
→ 204, o 410 si el token ya no es válido (el nodo lo olvida)
```

- `connect`: prioridad alta (push VoIP en iOS, prioridad alta en FCM). Lo dispara un sobre con una `offer` dirigido a ese dispositivo. El nodo DEBE retener el sobre hasta su `exp` (120 s como máximo) mientras espera a que el dispositivo abra sesión, y entregarlo con `deliver` en cuanto lo haga. Es la única retención de sobres permitida.
- `online`: prioridad baja, silencioso; puede no entregarse. Lo cubre el §12.4.

La pasarela no recibe ningún `uid`, `sid` ni contenido; solo un token y una palabra. Cualquiera PUEDE operar una pasarela para su propia compilación del cliente, porque las credenciales de push (FCM/APNs) están ligadas a la aplicación publicada. UnifiedPush es un transporte válido en Android sin la pasarela de nadie: en ese caso `push.gateway` es el endpoint UnifiedPush del dispositivo y el cuerpo es el mismo.

---

## 14. Errores

Las respuestas HTTP usan códigos estándar y un cuerpo `{ "error": "<código>", "detail": "<texto libre>" }`. Los mensajes de sesión usan `{ "t": "error", "id": "…", "code": "<código>" }`.

| code | HTTP | Significado |
|---|---|---|
| `invalid` | 400 | Objeto mal formado, falta un campo obligatorio o firma no válida |
| `expired` | 410 | Prueba, entrada, token o sobre caducado |
| `not_found` | 404 | El `uid` no existe en este nodo |
| `not_responsible` | 403 | El nodo no es responsable de ese `uid`; puede indicar quién lo es |
| `seq` | 409 | `seq` no es mayor que el almacenado |
| `unknown` | — | Destinatario desconocido o bloqueado (solo en sesión) |
| `offline` | — | Ningún dispositivo del destinatario tiene sesión ni push (solo en sesión) |
| `limit` | 429 | Cuota superada |
| `unauthorized` | 401 | Autenticación de sesión o de nodo no válida |
| `unavailable` | 503 | No se pudo contactar con el nodo responsable del destinatario o no respondió a tiempo; reintentar más tarde |

---

## 15. Límites

| Parámetro | Valor |
|---|---|
| Sobre: `exp` máximo / tamaño | 120 s / 16 KB |
| Entrada de dispositivo: `exp` máximo | 30 días; renovar a los 20 |
| Dispositivos por `uid` | 32 |
| `homes` / `endpoints` por entrada | 4 / 4 |
| Token de verificación | 15 minutos |
| Peticiones de verificación | 5 por dirección y hora |
| Vigilancias por vigilante / `ttl` | 256 / 7 días |
| Credenciales de relay | 1 hora |
| Relay por sesión | cuota del operador; recomendado 1 Mbit/s y 10 minutos |
| Ping de la sesión WebSocket | 30 s; el nodo cierra tras 90 s sin tráfico |
| Tabla de shards: `exp` | 7 días |
| Intento directo por `endpoints` | 3 s |
| Sondas ICE: intervalo / total | 50–200 ms / 30 s |
| `prange.count` | ≤ 20 |
| Keepalive del canal | 15–25 s; muerto tras 3 perdidos |
| Retransmisión del handshake (`noise`) | 1 s, 5 intentos |
| Renovación de claves (`noise`) | 120 s o 2⁶⁰ paquetes |
| Carga útil del paquete (`noise`) | ≤ 1200 bytes |
| Memoria de duplicados de sobres | 5 minutos |
| Petición entre nodos: conexión / total | 5 s / 10 s |
| Entre nodos: homes probados por entrada / cortacircuitos | 4 / 30 s |
| Cachés del nodo: registros validados / registros de nodo y asociaciones | 1 hora / 24 horas |
| Memoria de repeticiones entre nodos | 5 minutos |

---

## 16. Seguridad

**Modelo de identidad.** Controlar el email es controlar la cuenta. Es una decisión deliberada: es lo que permite entrar desde cualquier dispositivo solo con el email. Las consecuencias se mitigan en el cliente, no en el protocolo: un aviso al usuario y a sus contactos cuando aparece un dispositivo nuevo, y la posibilidad de revocarlo desde cualquier otro.

**Sin autoridad de certificación entre dispositivos.** Los dos perfiles autentican con claves que el directorio ya avaló: directamente en `noise`, mediante una huella transportada en señalización firmada en `webrtc`. Las únicas firmas del sistema están en los objetos del directorio y en la señalización, ambas verificadas una vez y cacheadas. No hay ninguna autoridad en la que confiar, que comprometer o a la que pagar.

**Qué ve un nodo.** El `uid` de origen y de destino de cada sobre que pasa por él, las vigilancias y qué dispositivos tienen sesión abierta. No ve el contenido de la señalización ni de la aplicación, ni puede alterarlos. Un relay ve paquetes cifrados y sus direcciones, nada más. Quien no quiera que un tercero vea ni siquiera eso, opera un nodo de dominio.

**Qué puede hacer un nodo malicioso.** Denegar el servicio a sus usuarios, responder falsamente `offline` o `unknown`, retener sobres o negarse a hacer de relay. No puede leer, modificar ni suplantar, porque todo va firmado o cifrado con claves que nunca salen de los dispositivos.

**Suplantación de direcciones.** En un dominio con entrada DNS requiere la clave privada del dominio. En el directorio público requiere engañar a un verificador aceptado por la federación. La fijación en el cliente convierte un cambio de `dom_pk` en una alerta y un dispositivo nuevo en un aviso.

**Abuso del handshake.** Un `handshake_init` le cuesta al receptor dos operaciones Diffie-Hellman. Un receptor bajo carga PUEDE limitar la frecuencia por dirección de origen; como el iniciador retransmite, las sesiones legítimas sobreviven. La marca de tiempo impide repetir un `handshake_init` capturado. DTLS tiene su propio mecanismo de cookies.

**Enumeración.** El `uid` es un hash sin sal, así que quien tenga una lista de direcciones de email puede comprobar cuáles están en el directorio público. Se acepta: el directorio no devuelve emails, solo claves, y los dominios con nodo propio no aparecen en él.

**Reenvío abierto.** Un nodo reenvía solo a URL que la resolución ha producido y que demuestran ser nodos (§9.4), nunca a una URL que un cliente o una entrada nombren por su cuenta, y acepta peticiones reenviadas solo de nodos que son home de los usuarios por los que hablan. Sin la primera regla un nodo sería un proxy hacia hosts arbitrarios, incluida su propia red privada; sin la segunda, cualquiera capaz de operar un nodo podría inundar a otro en nombre de usuarios inventados, y los avisos `online` se convertirían en un canal de spam de push.

**Spam de conexiones.** Una `offer` solo llega a un dispositivo con sesión abierta, y la aplicación decide si molesta al usuario. `block` corta el resto. Los nodos DEBERÍAN limitar la frecuencia de `connect` por origen.

**Metadatos de red.** Los candidatos ICE revelan direcciones IP al otro dispositivo (no al nodo, porque viajan sellados). Un usuario que no quiera revelar su IP a un contacto PUEDE configurar el cliente para ofrecer solo candidatos relay.

---

## 17. Identificadores registrados

| Elemento | Valor |
|---|---|
| Registro DNS | `_7ks.<dominio>` TXT |
| Well-known | `/.well-known/7ks` |
| Prefijo de la API | `/7ks/v1/` |
| Subprotocolo WebSocket | `7ksp.v1` |
| Puerto STUN / relay | 3478 (estándar); relay también en 443 TCP/TLS |
| Perfiles de transporte | `noise`, `webrtc` |
| Tipos de paquete `noise` | 0xC1–0xC4 (primer byte) |
| Prólogo Noise | `7ksp/1` |
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

**Convención de nombres.** La federación pública publica sus nodos de shard como nombres de host: el prefijo `p` del directorio público (`uid[0..2]`) se sirve en `https://<p en 4 dígitos hexadecimales en minúsculas>.7ks.app`, de `0000.7ks.app` a `ffff.7ks.app`. Un cliente o nodo de esta federación PUEDE derivar el nodo de un prefijo a partir del nombre sin descargar la tabla; la tabla sigue siendo la autoridad y se publica en la URL de arranque. Todos los nombres de un nodo son alias de su registro de nodo (`url`), y un registro obtenido en cualquiera de ellos se acepta para los demás con el mismo `nid`. Todos los dispositivos de una dirección viven en el nombre de su prefijo: los `homes` de las entradas públicas, `watcher_home` y el `host` de las sesiones lo nombran. Mover un prefijo a otro nodo es cambiar el registro DNS de ese nombre tras la replicación (§6, `GET /sync`); mientras tanto el nodo anterior responde `307` en lecturas y `403 not_responsible` en escrituras de ese prefijo (§9.1, §9.2).

## Anexo B. Ejemplos completos

### B.1 Llamada dentro del mismo dominio

A es `garacil@7kas.com`, dispositivo A1 (nativo, `noise` y `webrtc`), conectado al nodo de dominio N (`https://node.7kas.com`). B es `soporte@7kas.com`, dispositivos B1 (nativo) y B2 (navegador, solo `webrtc`), mismo nodo.

```
A1 → N   ws  {t:"resolve", id:"1", uid:<B>, domain:"7kas.com"}
N  → A1  ws  {t:"record", id:"1", record:{devices:[B1, B2]}}        // A1 valida la domainsig y fija las claves
A1       abre su socket UDP, consulta STUN, reúne candidatos ICE; también prepara una PeerConnection para B2
A1 → N   ws  {t:"connect", id:"2", env:{to:{uid:<B>, dev:B1}, from:{uid:<A>, dev:A1}, exp,
                                         body:sealed(B1, offer{sid, apps:["7ks-call/1"], transports:["noise","webrtc"], noise:{ice}, webrtc:{sdp}})}}
A1 → N   ws  {t:"connect", id:"3", env:{to:{uid:<B>, dev:B2}, …, body:sealed(B2, offer{sid, transports:["webrtc"], webrtc:{sdp}})}}
N  → B1  ws  {t:"deliver", env}          // B1 tiene sesión
N  → B2  ws  {t:"deliver", env}          // B2 tiene sesión en el navegador
B1, B2   abren el cuerpo, verifican la firma de A1; la aplicación 7ks-call/1 suena en los dos
B1 → N   ws  {t:"connect", env:{to:{uid:<A>, dev:A1}, body:sealed(A1, answer{sid, app:"7ks-call/1", transport:"noise", noise:{ice}})}}
N  → A1  ws  {t:"deliver", env}
A1 → N   ws  {t:"connect", env:{to:{uid:<B>, dev:B2}, body:sealed(B2, bye{sid, reason:"answered_elsewhere"})}}   // B2 deja de sonar
A1 ↔ B1  sondas de conectividad desde los dos lados a la vez; se valida un par; A1 lo elige
A1 → B1  handshake_init  (Noise_IK, clave estática = dev_dh_A1)
B1       comprueba que la clave estática es una dev_dh de A; responde
B1 → A1  handshake_resp
A1 ↔ B1  audio como datagramas; señalización de la llamada por el stream 1. N ya no interviene.
```

Si hubiera respondido antes B2, la sesión usaría `webrtc`: A1 aplica la respuesta SDP de B2, DTLS se completa, los dos comparan la huella remota con la recibida en la señalización firmada y el audio va por una pista WebRTC.

### B.2 Chat entre dominios distintos

A es `garacil@7kas.com` en N. C es `cliente@otra.com`; `otra.com` no tiene nodo propio y el home de C1 es el nodo público P.

```
A1 → N   ws  {t:"resolve", id:"1", uid:<C>, domain:"otra.com"}
N        DNS _7ks.otra.com → ninguno; prefijo = uid_C[0..2]; GET en un nodo de shard → registro con attestation
N  → A1  ws  {t:"record", id:"1", record:{devices:[C1 {homes:["https://p.example"]}]}}   // A1 valida la attestation contra verifiers
A1 → N   ws  {t:"connect", id:"2", env:{to:{uid:<C>, domain:"otra.com", dev:C1}, from:{uid:<A>, domain:"7kas.com", dev:A1}, exp, body:sealed(C1, offer{apps:["7ks-chat/1"], …})}}
N        C no es suyo; resuelve; POST https://p.example/7ks/v1/relay {env} con Authorization 7ks-node
P  → C1  ws  {t:"deliver", env}
C1 → P   ws  {t:"connect", env:{to:{uid:<A>, dev:A1}, body:sealed(A1, answer{…})}}
P        A no es suyo; resuelve 7kas.com → N; POST https://node.7kas.com/7ks/v1/relay {env}
N  → A1  ws  {t:"deliver", env}
A1 ↔ C1  ICE, handshake, stream 1 con los mensajes del chat.
```

### B.3 Mensaje a alguien que está apagado

Continuación de B.2, con C1 apagado:

```
A1 → N   connect → N: POST /relay en P → P: C1 no tiene sesión pero sí push registrado → push "connect" → sin respuesta en 120 s → P descarta el sobre
N  → A1  sent {devices:[C1]} … y A1 no recibe respuesta en 60 s → A1 envía bye cancel y guarda el mensaje
A1 → N   watch {C, ttl:604800} → N: POST /watch en P → P guarda Watch(A→C)
(dos días después) C1 arranca → auth en P → P: POST /online en N {uid:C, watcher_uid:A} → N → A1 online {C}
A1 → N   connect → … → canal directo → A1 entrega el mensaje que guardó.
```

## Anexo C. Streams fiables (perfil `noise`)

Las tramas de stream viajan dentro de paquetes `data` cuya carga útil empieza por `0x01`. Todos los enteros son big-endian.

```
frame_type(1) stream_id(4) …

0x01 OPEN    stream_id(4) window(4)                          abre un stream; window = bytes que el emisor puede recibir
0x02 DATA    stream_id(4) offset(8) len(2) bytes             datos en el desplazamiento offset
0x03 ACK     stream_id(4) cum(8) window(4) n(1) [from(8) to(8)]×n
                                                             recibido todo por debajo de cum; n rangos adicionales recibidos
0x04 FIN     stream_id(4) final_offset(8)                    no hay más datos después de final_offset
0x05 RESET   stream_id(4) code(4)                            abortar, descartar todo
```

Reglas:

- Los `stream_id` son pares cuando los abre el iniciador Noise e impares cuando los abre el receptor; el 0 está reservado. Un OPEN con un id ya en uso es un error de protocolo: RESET con `code` 1.
- `window` inicial recomendada: 65536 bytes. Un emisor NO DEBE tener en vuelo más de `window` bytes sin confirmar por stream. El receptor actualiza `window` en cada ACK.
- Un receptor envía un ACK como máximo cada 2 tramas DATA o cada 25 ms, lo que ocurra antes, e inmediatamente al detectar un hueco.
- Retransmisión: un rango DATA no confirmado dentro del tiempo de retransmisión (RTO) se reenvía. El RTO empieza en 300 ms y se estima a partir de los tiempos de ida y vuelta medidos como en la RFC 6298; mínimo 100 ms, máximo 10 s. Un rango que tres ACK consecutivos dan por perdido se reenvía en el acto.
- Se PUEDEN empaquetar varias tramas en un paquete siempre que la carga útil no supere 1200 bytes.
- Un stream se cierra cuando los dos lados han enviado FIN y están confirmados todos los datos hasta `final_offset`, o con RESET.
- El control de flujo es por stream; el control de congestión de la sesión DEBERÍA ser una ventana de incremento aditivo y reducción multiplicativa sobre todos los streams. Las implementaciones PUEDEN usar cualquier algoritmo estándar.

Esto es, a propósito, lo mínimo necesario para chat y transferencia de ficheros; no es un transporte de propósito general ni pretende competir con uno.

## Anexo D. Lista de comprobación de conformidad

Lo que una implementación DEBE cumplir, según su papel, para llamarse 7KSP/1. Cada punto remite a la sección que lo define.

**Todo cliente**
- Calcula el `uid` exactamente como en el §3.1 y reproduce los vectores del §3.2.
- Genera `dev_pk` y `dev_dh` en el dispositivo y nunca las exporta (§3).
- Valida las entradas de dispositivo según el §4.2 y descarta las que fallan (§4.3).
- Resuelve según el §7, DNS antes que directorio público, sin recurrir nunca al directorio público cuando el dominio es autoritativo.
- Fija las claves por TOFU y avisa de dispositivos nuevos y de cambios de `dom_pk` (§7.1).
- Se da de alta por enlace (§8) y renueva antes de la caducidad (§8.4).
- Abre una sesión con desafío firmado (§9.3) y la mantiene viva con `ping`.
- Sella y firma la señalización (§10.2) y valida la señalización recibida en el orden indicado.
- Envía una `offer` por dispositivo y `bye answered_elsewhere` al resto (§10.4).
- Respeta los temporizadores del §10.6.
- Ejecuta ICE completo, sobre un único socket, con sondas simultáneas y candidatos peer-reflexive (§11.2).
- Verifica la identidad del otro extremo al establecer el canal (§11.4 o §11.5) y aborta si no coincide.
- Reintenta lo pendiente al arrancar y al recuperar la red, y usa vigilancias (§12).
- Envía `domain` con `auth`, `resolve` y `watch` y en los dos extremos de cada sobre (§9.3, §10.1), y nunca la dirección en sí fuera del cuerpo sellado.

**Cliente nativo, además**: perfil `noise` completo (§11.4, Anexo C), `endpoints` si los hay (§11.7), comportamiento móvil (§11.10).
**Cliente web, además**: perfil `webrtc` (§11.5), anunciando solo ese perfil.

**Todo nodo**
- Expone las rutas de lectura sin autenticación y las de escritura con validación completa (§9.1, §9.2).
- Autentica las sesiones por desafío y comprueba la entrada del dispositivo (§9.3).
- Nunca encola sobres, salvo la retención del push (§10.1, §13).
- Entrega `connect` solo a dispositivos con sesión; responde `offline` y `unknown` según el §9.3.
- Reenvía a otros nodos exactamente una vez, solo a nodos verificados en direcciones públicas, autentica las peticiones entre nodos, las acepta solo de los homes de los usuarios por los que hablan y responde `unavailable` cuando no puede contactar con el otro nodo (§9.4).
- Almacena y procesa las vigilancias según el §12.
- Opera STUN (§11.2).
- Aplica los límites del §15.

**Nodo de dominio, además**: emite `domainsig` (§4.1, §8.7) y es verificador o delega en el procedimiento del dominio (§8).
**Nodo de shard, además**: mantiene la tabla firmada, replica sus prefijos, acepta escrituras solo de sus prefijos y rechaza los dominios con DNS (§5.2, §6).
**Verificador**: cumple el §8.3.
**Relay**: TURN estándar con cuotas y, cuando sea posible, TCP/TLS en el 443 (§11.2).
**Pasarela de push**: acepta `POST /7ks/v1/push` sin más datos que el token y el tipo (§13).

---

## Historial

- **0.6 (borrador, sin publicar)** — Contacto entre nodos completado: `domain` junto a cada `uid` que un nodo tiene que resolver (`auth`, `resolve`, `watch`, los dos extremos del sobre, el objeto de vigilancia); la autenticación entre nodos vincula al llamante con los usuarios por los que habla y la petición con el receptor (`url` en `Authorization`, `host` en el objeto firmado, memoria de repeticiones); reenvío solo a nodos verificados en direcciones públicas; `homes` comprobados al escribir; error `unavailable`; cachés, tiempos y cortacircuitos del nodo; convención de nombres de la federación pública (Anexo A).
- **0.5** — Reescritura para el máximo detalle y claridad: glosario, niveles de conformidad por papel, el recorrido de una conexión, algoritmos de validación y resolución en pseudocódigo, sealed box especificada, máquina de estados de la sesión, varios dispositivos, semántica exacta de `connect`, renovación y recuperación de la cuenta, explicación paso a paso del NAT, sección de móviles, ejemplos entre dominios y de entrega diferida, lista de comprobación de conformidad. Sin cambios en el formato de red respecto a la 0.4, salvo: `bye` lleva `reason`; `revoke` lleva `type`; subprotocolo WebSocket `7ksp.v1`; nonce del AEAD definido explícitamente.
- **0.4** — Dos perfiles de transporte bajo un mismo protocolo: `noise` (nativo) y `webrtc` (navegadores y nativos que quieran hablar con ellos), negociados por sesión con `transports` en las entradas de dispositivo y en `offer`/`answer`; garantías idénticas, ambos sin autoridades de certificación; primitivas de aplicación asignadas a cada perfil; los clientes web vuelven al alcance como clientes solo `webrtc`.
- **0.3** — Canal directo redefinido: solo UDP; handshake `Noise_IK` con las claves `dev_dh` de los dispositivos en lugar de TLS/DTLS; sin certificados ni SDP; requisitos completos de ICE; formato de paquetes y primitivas; streams fiables en el Anexo C; `endpoints`; `stun` y `roles.relay` en los registros de nodo.
- **0.2** — Definición completa: JSON con firmas JCS, API de nodo HTTPS/WebSocket, alta por enlace de email, pruebas por dispositivo, entrega diferida basada en vigilancias.
- **0.1** — Esquema inicial.
