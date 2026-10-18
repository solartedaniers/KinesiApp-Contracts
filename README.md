# KinesiApp Contracts

Este repositorio es la única fuente de verdad de los contratos de KinesiApp. Aquí defino en Protobuf todos los servicios y mensajes con los que se comunican el frontend, el backend y el servicio de IA.

Trabajo contract-first: primero diseño el `.proto`, después cada repo genera su código a partir de él y solo entonces implemento. Si un RPC no está aquí, no existe.

## Cómo está organizado

- `proto/kinesiapp/<dominio>/v<N>/`: un paquete versionado por dominio (`kinesiapp.system.v1`, `kinesiapp.auth.v1`, `kinesiapp.analysis.v1`, `kinesiapp.ai.v1`...).
- Uso [Buf](https://buf.build) para el lint y para detectar cambios incompatibles antes de publicar.
- Cada release es un tag semver (`v0.1.0`, `v0.2.0`...).

## Dominios

| Paquete | Servicios | Quién lo llama |
|---|---|---|
| `kinesiapp.common.v1` | Solo tipos compartidos | Todos |
| `kinesiapp.system.v1` | `SystemService` | Frontend → backend |
| `kinesiapp.auth.v1` | `AuthService` | Frontend → backend |
| `kinesiapp.account.v1` | `AccountService` | Frontend → backend |
| `kinesiapp.admin.v1` | `UserAdminService` | Frontend → backend |
| `kinesiapp.athlete.v1` | `AthleteProfileService`, `ManagedAthleteService` | Frontend → backend |
| `kinesiapp.team.v1` | `TeamService` | Frontend → backend |
| `kinesiapp.analysis.v1` | `AnalysisService` | Frontend → backend |
| `kinesiapp.chat.v1` | `ChatService` | Frontend → backend |
| `kinesiapp.ai.v1` | `MovementAnalysisService`, `AssistantService`, `AiLifecycleService` | Backend → IA (Envoy + gRPC-Web, con secreto compartido) |

Un dominio nunca importa mensajes de otro; solo de `common`. Así cualquiera puede desplegarse aparte.

## Convenciones

- **Errores**: status code estándar de gRPC más un `google.rpc.ErrorInfo` en los detalles, con `domain = "kinesiapp"` y `reason` igual al nombre del valor de `kinesiapp.common.v1.ErrorReason` sin el prefijo (`EMAIL_ALREADY_REGISTERED`).
- **Acceso**: cada RPC declara quién puede llamarlo con la opción `(kinesiapp.common.v1.access)`. Sin la opción, exige sesión con cualquier rol. El token viaja en la metadata `authorization: Bearer <token>`.
- **Streaming**: solo server streaming, porque gRPC-Web no admite client ni bidi streaming. Lo uso en el progreso del análisis y en las respuestas del chat.
- **Archivos**: nunca viajan en un mensaje. Un RPC entrega una URL firmada de Neon Storage, el navegador sube directo al bucket y otro RPC confirma la subida.
- **Salud**: el estándar `grpc.health.v1.Health`, que no se define aquí, más `AiLifecycleService.WarmUp` en la IA.

## Cómo lo consumen los demás repos

Cada repo (`KinesiApp-Backend`, `KinesiApp-Frontend`, `KinesiApp_IA`) agrega este repositorio como submódulo git en `contracts/`, fijado a un tag. El código generado se produce en el build de cada repo y nunca se versiona ni se edita a mano.

```sh
git submodule add https://github.com/solartedaniers/KinesiApp-Contracts.git contracts
git -C contracts checkout v0.2.0
```
